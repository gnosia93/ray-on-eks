이미지 전처리는 모델이 데이터를 더 잘 이해하도록 돕는 표준화(Standardization) 단계와 데이터의 양과 다양성을 늘리는 증강(Augmentation) 단계로 크게 나뉘어 진다. 

### 1. 기본 규격화 (Standardization) ###
모델 입력값의 형식을 맞추는 필수 단계이다. 
* 리사이징(Resizing): 모든 이미지를 일정한 크기(예: 224x224)로 통일.
* 정규화(Normalization): 픽셀 값(0~255)을 0~1 사이로 변환하거나, 평균과 표준편차를 사용하여 데이터 분포를 맞춤.
* 색상 공간 변환(Color Space Conversion): RGB 이미지를 흑백(Grayscale)으로 바꾸거나, 특정 채널만 추출. 

#### 참고 - 모델별 표준 고해상도 규격 ####
학습하려는 모델 아키텍처에 따라 권장되는 크기가 다르다.
* 299x299: Inception v3, Xception 모델의 표준 입력 크기.
* 336x336: 최근 멀티모달 모델(예: CLIP)이나 고성능 비전 트랜스포머(ViT)에서 자주 쓰이는 크기.
* 384x384: ViT(Vision Transformer)나 ConvNeXt의 대형 모델들이 미세 조정(Fine-tuning) 시 주로 채택하는 해상도.
* 448x448 또는 512x512: 객체 탐지(Object Detection)나 세그멘테이션처럼 작은 물체를 찾아내야 할 때 선호.
* 600x600: EfficientNet-B7과 같은 초대형 모델들이 극강의 정확도를 위해 사용하는 크기.

### 2. 품질 개선 및 노이즈 제거 ###
이미지의 특징을 더 명확하게 드러내기 위한 작업이다. 
* 노이즈 제거(Denoising): 가우시안 블러(Gaussian Blur)나 미디언 필터 등을 사용하여 불필요한 잡음을 제거.
* 대비 향상(Contrast Enhancement): 히스토그램 평활화(Histogram Equalization)를 통해 너무 어둡거나 밝은 이미지의 명암비를 조정.
* 이진화(Thresholding/Binarization): 이미지를 흑과 백 두 가지 색으로만 표현하여 객체의 형태를 단순화. 

### 3. 데이터 증강 (Augmentation) ###
학습 데이터가 부족할 때 기존 데이터를 변형시켜 모델의 일반화 성능을 높인다. 
* 기하학적 변형: 좌우/상하 반전(Flipping), 회전(Rotation), 크롭(Cropping), 아핀 변환(Affine Transform).
* 픽셀 레벨 변형: 밝기(Brightness) 조절, 채도(Saturation) 조정, 가우시안 노이즈 추가

## ViT 이미지 Augmentation ##
[preprocess.py]
```
import ray
import numpy as np
from torchvision import transforms

def preprocess_safe_center(batch):
    # 사물을 놓치지 않기 위한 안전한 전략
    transform = transforms.Compose([
        # 1. 짧은 축을 256으로 리사이즈 (비율 유지)
        transforms.Resize(256),       
        # 2. 중앙을 기준으로 224x224 자르기 (사물 보존 확률 극대화)
        transforms.CenterCrop(224),   
    ])
    
    # 처리 후 넘파이 배열(uint8)로 변환
    processed_images = [np.array(transform(img)) for img in batch["image"]]
    return {"image": processed_images}

# Ray 초기화 및 S3 읽기
# concurrency는 네트워크 상황에 따라 10~20 사이로 조절하세요.
ds = ray.data.read_images("s3://my-input-bucket/images/", concurrency=10)

# 분산 전처리 실행
# RandAugment는 학습 시 실시간으로 적용하기 위해 여기선 제외했습니다.
processed_ds = ds.map_batches(
    preprocess_safe_center,
    batch_size=128,
    compute=ray.data.ActorPoolStrategy(min_size=2, max_size=8)
)

# S3에 Parquet으로 저장
# 나중에 파이토치에서 읽기 가장 좋은 포맷입니다.
processed_ds.write_parquet("s3://my-output-bucket/preprocessed_data_224/")
```
아래 명령어로 ray 클러스터에 작업을 던진다.
```
# 로컬에서 실행 시
python preprocess_s3.py

# Ray 클러스터에 제출 시 (권장)
ray job submit --address http://<HEAD_NODE_IP>:8265 --working-dir . -- python preprocess.py
```


