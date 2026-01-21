이미지 전처리는 모델이 데이터를 더 잘 이해하도록 돕는 표준화(Standardization) 단계와 데이터의 양과 다양성을 늘리는 증강(Augmentation) 단계로 크게 나뉘어 진다. 

### 1. 기본 규격화 (Standardization) ###
모델 입력값의 형식을 맞추는 필수 단계이다. 
* 리사이징(Resizing): 모든 이미지를 일정한 크기(예: 224x224)로 통일.
* 정규화(Normalization): 픽셀 값(0255)을 0~1 사이로 변환하거나, 평균과 표준편차를 사용하여 데이터 분포를 맞춤.
* 색상 공간 변환(Color Space Conversion): RGB 이미지를 흑백(Grayscale)으로 바꾸거나, 특정 채널만 추출. 

### 2. 품질 개선 및 노이즈 제거 ###
이미지의 특징을 더 명확하게 드러내기 위한 작업이다. 
* 노이즈 제거(Denoising): 가우시안 블러(Gaussian Blur)나 미디언 필터 등을 사용하여 불필요한 잡음을 제거.
* 대비 향상(Contrast Enhancement): 히스토그램 평활화(Histogram Equalization)를 통해 너무 어둡거나 밝은 이미지의 명암비를 조정.
* 이진화(Thresholding/Binarization): 이미지를 흑과 백 두 가지 색으로만 표현하여 객체의 형태를 단순화. 

### 3. 데이터 증강 (Augmentation) ###
학습 데이터가 부족할 때 기존 데이터를 변형시켜 모델의 일반화 성능을 높인다. 
* 기하학적 변형: 좌우/상하 반전(Flipping), 회전(Rotation), 크롭(Cropping), 아핀 변환(Affine Transform).
* 픽셀 레벨 변형: 밝기(Brightness) 조절, 채도(Saturation) 조정, 가우시안 노이즈 추가
