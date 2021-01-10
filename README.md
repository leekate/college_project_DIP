# PROJECT: 
# Cephalogram 이미지 내 표시되어 있는 점들에 자동으로 Labeling하기


**cephalogram**
cephalogram 은 두개 안면 영역의 X- 선 이미지다. 두개골과 연조직의 프로파일 X-레이이며 어린이의 성장 측정, 턱의 치아 관계, 턱과 두개골의 관계 및 연조직과 치아와 턱의 관계를 평가하는 데 사용된다. 

![그림1](https://user-images.githubusercontent.com/46522501/104128837-2b8c6180-53ad-11eb-9f57-e4f126a7d34a.png)


# Processs:
## 1. 데이터 파악
9가지 데이터의 이미지 사이즈가 모두 다른 것 확인 -> 이미지 사이즈를 임의로 조절한 후 데이터를 다뤄야 한다 (700*1000)
이미지별 Luminance 차이 존재 확인

## 2. 프로그램 설계
  Color Picking으로 빨간점과 좌표를 추출
  Morphology로 점의 형태 보완
  빨간점을 기준으로 (20,10) 크기의 이미지를 크롭 
  Color Picking으로 글자를 추출해 Binary이미지로 변환
  Tesserrract OCR 사용으로 글자 인식

## 3. 시행 및 오류
#### 1. RGB 이미지를 HSV 이미지로 변형하고 HSV 컬러 범위를 이용해 검출하고자 하는 빨간점을 색출하기 위한 mask 만들기
8x8 커널을 활용해 만든 mask에 morphology closing 기법을 적용, binary image를 만들기
![다운로드](https://user-images.githubusercontent.com/46522501/104128588-ddc32980-53ab-11eb-82fa-7751923c1c15.png)


만든 mask의 좌표를 추출하기위해 contour를 적용하고 contour를 통해 나온 좌표들에 mean을 하여 해당 object의 center를 추정하였습니다.앞서 만든 binary image로 contour 찾기를 진행합니다. cv2.RETR_EXTERNAL를 이용해 이미지의 가장 바깥쪽인 contour만 추출하고 CHAIN_APPROX_NONE로 contour를 구성하는 모든 점을 저장한다.
이렇게 추출된 contour가 일정 갯수 미만이라면 빨간점이 모두 검출된 것이 아니므로 error 보이기.

**오류1. 원래 하나의 점인데 binary이미지를 만드는 과정에서 morphology close연산을 했음에도 하나의 object가 되지 못한 경우 발생
해결1. 원래 하나의 점인데 binary이미지를 만드는 과정에서 morphology 연산을 했음에도 하나의 object가 되지 못한 경우를 제거해야 한다
-x,y, 좌표의 평균값을 정리한 points 리스트를 이용해 최종 좌표를 추출하고
-distance 함수를 만들어 일정 거리 미만 떨어져 있는 점과 본인 점을 하나의 점으로 인식하고 그 점이 중복되어 리스트 안에 존재하지 않도록 set함수를 이용해 정리해주어야 한다.
-최종 리스트를 만들어 존재하는 점들의 최종 좌표를 추출한다.**


**오류2. 아래 그림처럼 모든 케이스에 동일한 처리를 해서는 Tesserract OCR을 취했을 때 각각의 이미지에서 최적의 깔끔한 처리를 할 수 없었다. 
해결2. 크롭한 영문 이미지를 모두 케이스별로 분류하여 간단한 모델을 사용해 딥러닝 학습을 시행하고 그중 가장 적합한 weight를 저장해 최종 단계에서 최적의 결과를 얻을 수 있도록 한다.**
![다운로드 (1)](https://user-images.githubusercontent.com/46522501/104128626-17943000-53ac-11eb-88d8-b1c3821e558a.png)



## 4. 최종 시행
![다운로드 (2)](https://user-images.githubusercontent.com/46522501/104128646-3d213980-53ac-11eb-91d0-03d92715f51f.png)
앞선 과정을 통해 추출한 points가 정확한지 확인해보았다.

다음으로,
점의 최종 좌표를 기준으로 위로 15픽셀, 오른쪽으로 29픽셀을 잘라 영단어만 포함된 이미지를 만들고 다루기 쉬운 numpy array 형태로 리턴할 수 있도록 함수를 만들어주었다.
(크롭 이미지의 크기는 수차례 경험 후 최적의 사이즈를 도출해낸 것이다. 직접 수치를 조절하며 수정해야 한다.)

## 5. 구현 및 결과 분석
 각 좌표와 prediction의 결과를 dictionary 형태로 매칭하여 반환하도록 한다.만들어진 dictionary들을 csv 파일에 작성하는 함수까지 모두 작성했으니 갖지고 있는 Cephalogram 데이터의 모든 케이스에 적용시킨다.
 
 ![다운로드 (3)](https://user-images.githubusercontent.com/46522501/104128685-680b8d80-53ac-11eb-8352-1f2add9cbeb9.png)
전체 점의 갯수 (총 17*9=136) 153개 중 1가지 좌표를 제외한 152개가 일치하는 것을 확인할 수 있었다. 
하나의 좌표가 차이가 나는데 이건 딥러닝 Classification의 문제가 아니라 crop된 이미지 중 B가 적힌 이미지가 여러개인 경우에 해당한다.


실제로 해당 케이스를 확인해본 결과 아래 사진과 같았다. 크롭된 이미지에 B가 두번 포함되어 버린 경우였다. 이를 해결하기 위해 X의 좌표가 더 큰 값을 택하기로 했다.

![다운로드 (4)](https://user-images.githubusercontent.com/46522501/104128721-8a9da680-53ac-11eb-84f2-1f3c9b548769.png)


수정 후 모든 경우가 일치하는 것을 확인할 수 있다.
![다운로드 (5)](https://user-images.githubusercontent.com/46522501/104128748-a5701b00-53ac-11eb-9091-1dfe9584b2c5.png)




