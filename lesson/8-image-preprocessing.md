이미지 전처리는 모델이 데이터를 더 잘 이해하도록 돕는 표준화(Standardization) 단계와 데이터의 양과 다양성을 늘리는 증강(Augmentation) 단계로 크게 나뉘어 진다. 

## 전처리와 증강 ##
CPU 에서 전처리 및 증강 -가장 정석적이고 안전한 전략입니다. 특히 데이터셋이 아주 크거나, 학습 중에 CPU가 노는 것이 아깝다면 전처리(Preprocessing)와 증강(Augmentation)을 분리하는 게 효율적입니다.
다만, 이때 고려해야 할 한 가지 핵심 포인트가 있습니다.

#### 1. 전처리와 증강의 구분 ####
* CPU 노드에서 미리 하기 (Offline): 리사이즈(Resize), 노이즈 제거, 데이터 정규화 등 모든 이미지에 동일하게 적용되는 작업은 미리 처리해서 저장해두는 게 압도적으로 유리합니다. Albumentations 같은 도구를 CPU 노드에서 돌려 결과물을 저장해두면 GPU 로드 시 시간을 대폭 아낍니다.
* GPU 학습 로드 시 하기 (Online): 매 에폭마다 이미지를 무작위로 비틀거나(Random Flip, Rotation) 색상을 바꾸는 '확률적 증강'은 학습 직전에 수행해야 모델의 성능(일반화)이 올라갑니다.

#### 2. 이 방식의 최대 장점 ####
* GPU 연산 극대화: GPU는 복잡한 전처리 계산을 신경 쓸 필요 없이 오직 학습(Backpropagation)에만 모든 자원을 쏟아부을 수 있습니다.
* 스토리지 절약: 전처리가 끝난 데이터는 보통 용량이 최적화되어 있어, 네트워크를 통해 GPU 노드로 데이터를 전송할 때 대역폭(I/O) 병목을 줄여줍니다.

#### 3. 주의할 점: '데이터 로딩 속도' ####
CPU 노드에서 전처리를 마쳤더라도, GPU 학습 시 데이터를 하드디스크(HDD/SSD)에서 읽어오는 속도가 느리면 GPU가 대기하게 됩니다. 이를 해결하려면 PyTorch DataLoader의 num_workers 옵션을 CPU 코어 수에 맞춰 넉넉히 설정하고, pin_memory=True를 사용해 GPU 전송 속도를 높여야 합니다.


### 해상도별 하드웨어 선택 가이드 ###
#### 저해상도 (224 x 224 ~ 512 x 512) ####
* CPU 유리: 개별 연산량이 적어 GPU로 데이터를 옮기는 시간(오버헤드)이 더 아깝습니다. Albumentations 같은 CPU 라이브러리로도 충분히 빠릅니다.
#### 고해상도 (1024 x 1024 ~ 2048 x 2048) ####
* 혼합 영역: CPU 코어 수가 많다면 CPU에서 처리해도 되지만, 학습 속도가 느려진다면 GPU 가속을 고민해야 합니다.
#### 초고해상도 (4000 x 4000 이상, 4K+): ####
* GPU 압도적 유리: 픽셀 수가 천만 개를 넘어가면 CPU는 디코딩과 증강에서 심각한 병목을 일으킵니다. 이때는 NVIDIA DALI나 Kornia를 써서 GPU 내에서 직접 처리하는 것이 최대 수십 배 빠릅니다.
---
2. 고정적인 기하학적/색상 변환
데이터 증강(Augmentation)이 아닌, 모든 데이터에 공통적으로 적용되어야 하는 표준화 작업은 미리 해두는 것이 좋습니다. 
배경 제거 및 크로핑 (Clipping): 이미지 내에서 관심 영역(ROI)만 추출하거나 불필요한 배경(여백)을 제거하는 작업은 모델의 집중도를 높입니다.
그레이스케일 변환: 색상 정보가 필요 없는 데이터셋(예: OCR, 지문 등)은 미리 1채널로 변환하여 데이터 크기를 1/3로 줄일 수 있습니다.
색상 공간 변환: RGB를 YCbCr이나 Lab 컬러 공간으로 변환해야 할 경우, 수식 계산이 필요하므로 미리 처리해 두면 유리합니다. 
3. 통계적 정제 및 라벨링 관리
데이터의 질을 높이는 클리닝 작업은 CPU의 논리 연산 능력을 적극 활용해야 합니다. 
이상치(Outlier) 및 손상 파일 제거: 데이터 로딩 중 에러를 유발하는 깨진 파일이나, 값이 튀는 이미지를 사전에 필터링합니다.
데이터 셔플링 (Shuffling): 대규모 데이터셋의 경우 인덱스를 미리 섞어서 저장해 두면 학습 시 매번 셔플하는 부하를 줄일 수 있습니다.
라벨 인코딩: 범주형(Categorical) 텍스트 라벨을 숫자 인덱스(Integer)나 원-핫 인코딩 형식으로 미리 매핑해 두는 작업입니다. 
4. 특징 추출 (Feature Extraction)
딥러닝 모델 앞단에서 고정된 연산이 필요한 경우입니다. 
엣지 검출 (Sobel, Canny): 특정 모델 구조에서 경계선 정보를 추가 입력으로 쓴다면 미리 추출해 둡니다.
히스토그램 평활화 (Histogram Equalization): 이미지의 대비를 전반적으로 높여 조명 차이를 극복해야 할 때 유용합니다. 
이 작업들은 한 번만 고생하면 학습 시 GPU가 데이터 대기 없이(Starvation 방지) 풀가동되도록 도와줍니다. 



---
#### 1. 기본 규격화 (Standardization) ####
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

#### 2. 품질 개선 및 노이즈 제거 ####
이미지의 특징을 더 명확하게 드러내기 위한 작업이다. 
* 노이즈 제거(Denoising): 가우시안 블러(Gaussian Blur)나 미디언 필터 등을 사용하여 불필요한 잡음을 제거.
* 대비 향상(Contrast Enhancement): 히스토그램 평활화(Histogram Equalization)를 통해 너무 어둡거나 밝은 이미지의 명암비를 조정.
* 이진화(Thresholding/Binarization): 이미지를 흑과 백 두 가지 색으로만 표현하여 객체의 형태를 단순화. 

#### 3. 데이터 증강 (Augmentation) ####
학습 데이터가 부족할 때 기존 데이터를 변형시켜 모델의 일반화 성능을 높인다. 
* 기하학적 변형: 좌우/상하 반전(Flipping), 회전(Rotation), 크롭(Cropping), 아핀 변환(Affine Transform).
* 픽셀 레벨 변형: 밝기(Brightness) 조절, 채도(Saturation) 조정, 가우시안 노이즈 추가

## 이미지 전처리 ##

### 1. 이미지 수집하기 ###
* https://www.robots.ox.ac.uk/~vgg/data/pets/
```
import tarfile
import requests
import boto3
import io
from concurrent.futures import ThreadPoolExecutor

# --- 설정 구간 ---
S3_BUCKET_NAME = "your-my-input-bucket"  # 본인의 버킷명으로 수정
S3_PREFIX = "images/"                    # 저장할 폴더 경로
PET_DATA_URL = "https://www.robots.ox.ac.uk"
# ----------------

s3_client = boto3.client('s3')

def upload_to_s3(args):
    filename, file_content = args
    s3_key = f"{S3_PREFIX}{filename}"
    s3_client.put_object(Bucket=S3_BUCKET_NAME, Key=s3_key, Body=file_content)
    print(f"Uploaded: {filename}")

def main():
    print("다운로드 시작... (시간이 다소 소요될 수 있습니다)")
    response = requests.get(PET_DATA_URL, stream=True)
    
    # 메모리에서 타르 파일 읽기
    with tarfile.open(fileobj=io.BytesIO(response.content), mode="r:gz") as tar:
        # JPG 파일만 필터링
        jpg_files = [m for m in tar.getmembers() if m.name.endswith(".jpg")]
        print(f"총 {len(jpg_files)}개의 이미지를 발견했습니다. 업로드 시작...")

        # 병렬 업로드를 위한 데이터 준비 (파일객체 추출)
        upload_tasks = []
        for member in jpg_files:
            f = tar.extractfile(member)
            if f:
                # 파일명만 추출 (경로 제외)
                clean_name = member.name.split("/")[-1]
                upload_tasks.append((clean_name, f.read()))

        # ThreadPool을 사용하여 S3로 고속 병렬 업로드
        with ThreadPoolExecutor(max_size=10) as executor:
            executor.map(upload_to_s3, upload_tasks)

if __name__ == "__main__":
    main()
```

```
python upload_pets_to_s3.py
```

### 2. 데이터 증강 (Augmentation) ###
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

#### 참고 - 클러스터 지정 ####
* 기본 동작: 로컬(Local) Ray 클러스터
만약 명령어를 실행하는 현재 머신에 Ray 클러스터가 이미 실행 중이라면(ray start --head 명령어로 띄워져 있는 상태), ray job submit은 기본적으로 자신의 로컬 머신(localhost:8265)에 떠 있는 클러스터 사용한다.

* 클러스터 지정: --address 옵션 사용
원격 Ray 클러스터(예: AWS EC2 인스턴스 여러 대로 구성된 클러스터)에 작업을 보내고 싶다면, Head Node의 주소를 명시적으로 지정해야 한다.


