```python
import atexit
import json
import logging
import threading
import torch
from typing import Callable
from wanx_infer import WanxInfer
import io
from base64 import encodebytes
from kafka import KafkaConsumer, KafkaProducer
from PIL import Image
import os
import shutil

wi = WanxInfer(True)
wi.initialize('/build/models/wanx_base/wanx_v1.3.1_230726')
logger = logging.getLogger(__name__)

class WanxConsumer:

    def __init__(self,
                 handler: Callable[[str, dict], any],
                 request_topic: str,
                 response_topic: str,
                 props: dict):
        super().__init__()
        assert handler
        self._running = False
        consumer_props = props.copy()
        if 'max_request_size' in consumer_props: del consumer_props['max_request_size']
        self._consumer = KafkaConsumer(request_topic, **consumer_props)
        #self._consumer = KafkaConsumer(request_topic, **props)
        self._thread = threading.Thread(target=self._listen)
        self._response_topic = response_topic
        self._handler = handler
        producer_props = props.copy()
        if 'max_poll_records' in producer_props: del producer_props['max_poll_records']
        if 'group_id' in producer_props: del producer_props['group_id']
        if 'auto_offset_reset' in producer_props: del producer_props['auto_offset_reset']
        if 'enable_auto_commit' in producer_props: del producer_props['enable_auto_commit']
        self._producer = KafkaProducer(
            **producer_props
        )

    def _handle(self, msg):
        key = msg.key.decode('utf-8')
        value = msg.value.decode('utf-8')
        logger.info(f"key={key},value={value}")
        print(f"key={key},value={value}")
        body = json.loads(value)
        try:
            result = self._handler(key, body)
            if type(result) is not str:
                result = json.dumps(result)
            print(len(result))
            print(self._response_topic)
            print(msg.key)
            self._producer.send(self._response_topic, value=result.encode(), key=msg.key)
            #self._consumer.commit()
        except Exception as e:
            logger.error("error caught during processing request: {}".format(e))

    def _listen(self):
        self._consumer.poll()
        print("message")
        for message in self._consumer:
            print(str(message))
            self._handle(message)

    def start(self):
        self._running = True
        self._thread.start()
        logger.info("Consumer started")

    def stop(self):
        self._running = False
        self._thread.join()
        self._producer.close()
        self._consumer.close()
        logger.info("Consumer stopped")

def transform_invert(img):
    # Tensor -> PIL.Image
    low = float(img.min())
    high = float(img.max())
    img.sub_(low).div_(max(high - low, 1e-5))  # (img - low)/(high-low)
    ndarr = img.mul(255).add_(0.5).clamp_(0, 255).permute(0, 2, 3, 1).to("cpu", torch.uint8).numpy()
    return ndarr

def encode_pil_image(pil_img):
    byte_arr = io.BytesIO()
    pil_img.save(byte_arr, format='PNG')  # convert the PIL image to byte array
    encoded_img = encodebytes(byte_arr.getvalue()).decode('ascii')  # encode as base64
    return encoded_img


def handle_task():
    # 设置日志级别
    logging.basicConfig(level=logging.INFO)

    cli_args = os.environ.get("CLI_ARGS")
    # 分割字符串
    split_string = cli_args.split("--")

    # 初始化结果字典
    result_dict = {}

    # 解析键值对并生成字典
    for item in split_string:
        if "=" in item:
            key, value = item.split("=")
            result_dict[key.strip()] = value.strip()

    conf = {
        'bootstrap_servers': result_dict["kafka-bootstrap-servers"],
        'group_id': result_dict["kafka-group-id"],
        # 'client.id': cmd_opts.kafka_client_id,
        'auto_offset_reset': result_dict.get("kafka-auto-offset-reset", "latest"),
        'security_protocol': result_dict["kafka-security-protocol"],
        'sasl_mechanism': result_dict.get("kafka-sasl-mechanism", "PLAIN"),
        'sasl_plain_username': result_dict["kafka-sasl-username"],
        'sasl_plain_password': result_dict["kafka-sasl-password"],
        'ssl_cafile': result_dict["kafka-cafile"],
        # 'enable_auto_commit': result_dict["kafka-consumer-enable-auto-commit"],
        'enable_auto_commit': True,
        'max_poll_records': 1,
        'ssl_check_hostname': False,
        'max_request_size': 10485760
    }
    logger.info('Kafka config props:')
    for k, v in conf.items():
        logger.info(f'{k}={v}')
        print(f'{k}={v}')

    def initialize_consumer(wi: WanxInfer):
        print("initialize_consumer start")

        def handle_wanx_message(key: str, body: dict):
            try:
                method, task_id = key.split("#")
                # 软链接 checkpoint lora
                checkpoint_file_name = body.get("checkpointFileName")
                links_to_delete = []
                if checkpoint_file_name is not None:
                    if not os.path.exists("/build/models/wanx_checkpoint"):
                        os.makedirs("/build/models/wanx_checkpoint")
                    source_file_dir = "/build/models/wanx_checkpoint-global/{}"
                    target_file_dir = "/build/models/wanx_checkpoint/{}"
                    for key, value in body["checkpointFileName"].items():
                        source_path = source_file_dir.format(key)
                        target_path = target_file_dir.format(value)
                        if not os.path.exists(source_path):
                            return {
                                "errorMessage": "No such file or directory, source_path = {}".format(source_path)
                            }
                        if not os.path.exists(target_path):
                            os.symlink(source_path, target_path)
                            links_to_delete.append(target_path)
                            logger.info("checkpoint symlinked, source: %s, Target: %s", source_path, target_path)
                            print(f"checkpoint symlinked, source: {source_path}, Target: {target_path}")
                        else:
                            logger.warning("checkpoint symlink already exists, Source: %s, Target: %s", source_path,
                                       target_path)
                            print(f"checkpoint symlink already exists, source: {source_path}, Target: {target_path}")
                else:
                    logger.error("checkpoint is must!")
                # 从body中获取loraFileName，建立lora软连接
                lora_file_name = body.get("loraFileName")
                if lora_file_name is not None:
                    if not os.path.exists("/build/models/wanx_lora"):
                        os.makedirs("/build/models/wanx_lora")
                    source_file_dir = "/build/models/wanx_lora-global/{}"
                    target_file_dir = "/build/models/wanx_lora/{}"
                    for key, value in body["loraFileName"].items():
                        source_path = source_file_dir.format(key)
                        target_path = target_file_dir.format(value)
                        if not os.path.exists(source_path):
                            return {
                                "errorMessage": "No such file or directory, source_path = {}".format(source_path)
                            }
                        if not os.path.exists(target_path):
                            os.symlink(source_path, target_path)
                            links_to_delete.append(target_path)
                            logger.info("lora symlinked, source: %s, Target: %s", source_path, target_path)
                            print(f"lora symlinked, source: {source_path}, Target: {target_path}")
                        else:
                            logger.warning("lora symlink already exists, Source: %s, Target: %s", source_path, target_path)
                            print(f"lora symlink already exists, source: {source_path}, Target: {target_path}")
            except Exception as e:
                logger.error("os symlink fail key={}".format(key))

            try:
                indata = {
                    'batch': body['batch'],
                    'checkpoint_file_name': body['checkpointFileName'],
                    'load_model': body['loadModel'],
                    'negative_prompt': body['negativePrompt'],
                    'prompt': body['prompt'],
                    'seed': body['seed'],
                    'steps': body['steps'],
                    'unet_path': body['unetPath'],
                    'use_lora': body['useLora'],
                    'lora': body['lora'],
                    'wanx_infer': body['wanxInfer'],
                    'version': body['version']
                }
                if method == "Text2Image":
                    res = wi.process(indata)
                    np_imgs = transform_invert(res)
                    pil_imgs = [Image.fromarray(image) for image in np_imgs]
                    encoded_images = []
                    for i in range(body['batch']):
                        encoded_images.append(encode_pil_image(pil_imgs[i]))
                    data = {"images": encoded_images}
                    return {
                        "request": body,
                        "response": data
                    }
                if method == "Image2Image":
                    res = wi.process(body)
                    np_imgs = transform_invert(res)
                    pil_imgs = [Image.fromarray(image) for image in np_imgs]
                    encoded_images = []
                    for i in range(body['batch']):
                        encoded_images.append(encode_pil_image(pil_imgs[i]))
                    data = {'images': encoded_images}
                    return {
                        "request": body,
                        "response": data
                    }
            except Exception as e:
                shutil.rmtree("/build/models/wanx_lora")
                shutil.rmtree("/build/models/wanx_checkpoint")
                logger.error("unknown api for handler: key={}".format(key))
                logger.error("error messaeg: {}".format(e))
                return {
                    "errorMessage": str(e)
                }
            finally:
                #清除软链接
                for link in links_to_delete:
                    if os.path.exists(link):
                        os.unlink(link)
                        logger.info("unlink successful,targetPath: %s !",link)
                        print(f"unlink successful,targetPath: {link} !")

        consumer = WanxConsumer(
            handle_wanx_message,
            result_dict["kafka-request-topic"],
            result_dict["kafka-response-topic"],
            conf
        )
        consumer.start()
        atexit.register(consumer.stop)
    initialize_consumer(wi)

```

