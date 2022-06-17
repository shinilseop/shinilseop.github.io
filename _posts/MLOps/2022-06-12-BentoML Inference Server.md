---
title:  "BentoML Inference Server (간단한 인퍼런스 서버 with Docker)"
excerpt: "BentoML Inference Server Example"

categories:
  - MLOps
tags:
  - [MLOps, BentoML, Docker, Inference Server]

toc: true
toc_sticky: true
 
date: 2022-06-18
last_modified_at: 2022-06-18

---

# BentoML

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled.png)

BentoML simplifies ML model deployment and serves your models at production scale.

## **Feature Highlights**

✨ Model Serving the way you need it

- **Online serving** via REST API or gRPC
- **Offline scoring** on batch datasets with Apache Spark, or Dask.
- **Stream serving** with Kafka, Beam, and Flink

🍱 Easily go from training to model serving in production

- All common ML libraries natively supported! - Tensorflow, PyTorch, XGBoost, Scikit-Learn and many more
- Standard `.bento` format for packaging code, models and dependencies for easy versioning and deployment
- Automatically setup Dockerfile, CUDA and python packages for model serving
- Designed to work with any training pipeline or experimentation platform

🐍 Python-first, scales with powerful optimizations

- Scale model inference workers separately from business logic and feature processing code
- Adaptive batching dynamically groups inference requests for optimal performance
- Inference graphs with multiple models automatically orchestrated with Yatai on Kubernetes

🚢 Deployment workflow made for production

- 🐳 Automatically generate docker images for production deployment
- [🦄️ Yatai](https://github.com/bentoml/yatai): Model Deployment at scale on Kubernetes
- [🚀 bentoctl](https://github.com/bentoml/bentoctl): Fast model deployment on any cloud platform

위의 내용은 BentoML 공식 깃허브에서 제공하고 있는 내용이다.

BentoML은 머신러닝 모델을 만들고 난 후 서빙을 하는 과정에 있어서 필요한 기능들을 간단하게 사용할 수 있도록 프로덕션 레벨에서 제공하는 프레임워크이다.

훈련되어있는 모델을 가지고 도커에 인퍼런스 서버를 띄워보자

## 환경 구성

### Anaconda miniforge 설치

모듈 설치를 진행하기 전에 파이썬 환경을 분리하기 위해 아나콘다를 설치해서 가상환경을 구성했다.

[https://github.com/conda-forge/miniforge](https://github.com/conda-forge/miniforge)

위의 miniforge 깃 허브에 접근해서 mac os 중 arm64 아키텍처로 구성된걸로 다운해서 설치했다.

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled%201.png)

```bash
# 설치
sh ~/Downloads/Miniforge3-MacOSX-arm64.sh

# 환경변수 넣기
source ~/miniforge3/bin/activate
```

설치하고 아나콘다 환경을 새로 하나 생성했다.

```bash
conda create --name bentoml python=3.8

conda activate bentoml
```

### BentoML + 필요한 라이브러리 설치

```bash
pip install bentoml
pip install transformers # 사용할꺼라서...
pip install pytorch torchvision torchaudio # 얘도 마찬가지로 사용예정
```

기본적으로 bentoml 을 설치한 후 transformers와 pytorch 라이브러리를 설치했다.

이후 예제 코드를 돌리는데 bentoml을 설치하면서 종속성으로 설치된 grpc 모듈을 계속 가져오지 못하는 오류가 발생했다.

구글링하다보니 아마 Apple Silicon Mac m1 에서 발생하는 오류인거같은데 환경을 실행한 가상환경으로 못잡고 기본 파이썬으로 자꾸 잡으려고하는것 같다. 

해결하기 위한 다양한 방법들이 제시되어 있었는데

```bash
export GRPC_PYTHON_BUILD_SYSTEM_OPENSSL=1
export GRPC_PYTHON_BUILD_SYSTEM_ZLIB=1
```

위와 같이 환경 변수를 설정하고 다시 grpc를 설치하라는 내용이어서 따라 진행했지만 해결은 안됬다.

두번째로 다음의 pip install을 통해서 grpc를 다시 설치하는 것이었다.

`pip install --no-binary :all: grpcio --ignore-installed`

해결 안됬다..

마지막으로 conda의 grpc를 이용해서 설치하는 방법을 찾아서 진행했다

`conda install -c conda-forge grpcio`

위의 명령어를 통해서 다시 설치했더니 진행이 됬다 !~~ 👍

### 예제 코드

```python
# bentoml-service.py

import bentoml
from bentoml.adapters import JsonInput
from bentoml.frameworks.transformers import TransformersModelArtifact
import numpy as np
from transformers import AutoModelForSequenceClassification, AutoTokenizer

@bentoml.env(pip_packages=["transformers==4.11.0", "torch==1.7.0"])
@bentoml.artifacts([TransformersModelArtifact("transformer")])
class TransformerService(bentoml.BentoService):

    LABELS = {
        0: "부정",
        1: "긍정"
    }

    @bentoml.api(input=JsonInput(), batch=False)
    def predict(self, parsed_json):
        src_text = parsed_json.get("text")

        model = self.artifacts.transformer.get("model")
        tokenizer = self.artifacts.transformer.get("tokenizer")

        # Pre-processing
        input_ids = tokenizer.encode(src_text, return_tensors="pt")

        # Model forward
        output = model(input_ids)

        # Post-processing
        preds = output.logits.detach().cpu().numpy()
        preds = np.argmax(preds, axis=1)

        return {
            "result": self.LABELS[preds[0]]
        }

if __name__ == "__main__":
    ts = TransformerService()

    model_name = "monologg/koelectra-small-finetuned-nsmc"
    model = AutoModelForSequenceClassification.from_pretrained("monologg/koelectra-small-finetuned-nsmc")
    tokenizer = AutoTokenizer.from_pretrained("monologg/koelectra-small-finetuned-nsmc")
    artifact = {"model": model, "tokenizer": tokenizer}

    ts.pack("transformer", artifact)
    saved_path = ts.save()
```

위의 예제 코드는 영화 리뷰를 입력받아 결과를 내도록 pre-train 되어있는 모델을 사용해서 인퍼런스 서버로 구성하는 예제이다.

파이썬 파일을 작성한 후 bentoml bundle을 만들고 dockerize를 해서 도커에서 실행해보았다.

```python
python bentoml_service.py  # bentoml bundle 만들기
bentoml containerize TransformerService:latest -t tf_service:v1  # dockerize

docker run -p 5001:5000 tf_service:v1
```

처음에 5000번 포트를 사용하려했으나 자꾸 누가 사용중이라 확인해보니 ControlCE라는 job이 실행중이어서 5001번으로 진행했다. (나중에 알고보니 mac airplay란다…)

실행해서 접근해보면 다음과 같은 api를 설명하는 사이트를 볼 수 있다.

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled%202.png)

predict를 위해서는 http://0.0.0.0:5001/predict 를 POST로 input을 json으로 담아서 보내면 된다.

위의 웹페이지에서도 보내는 기능을 제공하는데 app의 predict에서 가능하다

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled%203.png)

Try it out 버튼을 누르고

위에 예제 코드에서 json 데이터 중에 text 키를 파싱해서 predict 하도록 만들었으므로 다음과 같이 데이터를 넣고 execute 버튼을 누르면 api 를 쏜다

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled%204.png)

그러면 아래와 같은 응답이 나타나게 된다.

![Untitled](../../assets/images/AI/MLOps/bentoml%20inference%20server/Untitled%205.png)

`너무 별로다...` 라는 데이터를 추론해본 결과 `부정` 이 도출된걸보아 잘 되었다.