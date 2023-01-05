Faster R-CNN: Towards Real-Time Object Detection with Region Proposal Networks
==============================================================================
최근의 State-of-the-art object detection method에서 region proposals 알고리즘이 차지하는 비중은 크다. SPP-Net과 Fast R-CNN에 이어서 새로운 Region Proposal Network(RPN)이라는 새로운 region proposals method을 소개한다. RPN은 Fully Convolutional Network형태로 각각의 위치에서 objectness scores와 object bounds를 동시에 예측한다. RPN을 통해 어디를 중점적으로 봐야하는지 attention의 개념을 적용해 뒤의 network에게 알려줄 수 있다. 

## Introduction
R-CNN 이 Region based CNN인 것처럼 최근 Region proposal은 Object detection 분야에서 생각보다 중요하다. 기존의 selective search와 같은 region proposal 알고리즘도 휼륭한 성과를 거두었지만 느리다는 단점이 존재한다. 그리고 selective search가 CPU로 작동하기 때문에 속도가 매우 느리고 GPU로 작동할 수 있는 알고리즘이 필요하다. 이러한 배경으로 등장한 것이 RPN이다. 마찬가지로 convolutional layer를 공유함으로써, 즉 한번의 convolutional을 통해 계산시간을 많이 단축할 수 있게 되었다. 위에서 살펴보았듯이 FCN구조이기 때문에 End-to-End 방식으로 학습시키는 것이 가능하다.

## Faster R-CNN
Faster R-CNN은 전체적으로 2가지 부분으로 나누어 진다. 첫 번째 모듈은 region proposal을 구하는 network, 이를 이용하는 Fast R-CNN detector이다. Faster R-CNN은 detection을 위한 single network로 여기서 RPN은 attention을 구하는 역할을 담당한다. 말그대로 뒤에 detector에게 "여기를 집중적으로 봐" 라고 하는 것과 같은 개념이다. 전체적인 구조는 다음과 같다. 

![Faster R-CNN](./img/Faster%20R-CNN.png)

## Dense Sampling

![dense_sampling](./img/dense_sampling.png)

selective search 방식을 사용하지 않을 경우 사용할 수 있는 방법은 무엇이 있을까? 한 가지 방법으로는 위의 얼룩말 이미지 800 x 800 을 몇 개의 grid로 나눈다. 여기서는 총 8칸으로 나누어서 64개의 grid cell을 구할 수 있다. 이 grid cell을 bouding box로 간주한다. 따라서 원본이미지에 64개의 bounding box가 생성된다는 의미이다. 하지만 이미지에서 물체의 크기는 매우 다양하기 때문에 동일한 크기로 bbox를 만드는 것은 적절하지 못하다. 따라서 저자들은 Anchor라는 개념을 소개한다. 

## Anchor Box

사전에 정한 scale값과 aspect ratio값을 사용하여 미리 anchor box라는 것을 만든다. 이는 앞의 Dense sampling 방식의 문제점을 해결 할 수 있다. 논문에서는 각각을 3으로 지정해 9개의 anchor box를 생성했다. 그런데 정확히 anchor 가 무엇을 하는지 감이 안 잡혀 다음 그림을 참조했다.

![anchor](./img/anchor.png)

먼저 anchor는 "정박하다"란 뜻으로 무언가를 고정시키는 뜻이다. anchor box는 여러 개의 grid로 나눈 이미지에서 각 grid cell의 중심에 anchor box들을 생성한다. 이때 생성되는 box의 개수는 다음과 같다. 위의 그림에서 입력 이미지가 800 x 600이고 grid cell의 높이와 너비를 16이라고 할 때 생성되는 cell의 개수는 800 / 16 * 600 / 16 이런 식으로 계산된다. 따라서 여기에 anchor box의 개수를 곱해주면 총 17100개의 anchor box가 생기는 것이다. 그림의 중앙 부분으로 갈수록 겹쳐지는 anchor box가 많은 것을 확인할 수 있다. 따라서 더 다양한 크기의 객체를 detection 하는 것이 가능해졌다.

입릭 이미지에 대하여 convolution을 수행할 때 K개의 region proposal을 동시에 얻어내는 것이다. 따라서 reg layer의 결과값으로는 4k개의 값이 나오고 cl layer의 결과값으로는 2k개의 결과가 나오는 것이다. 2k라 한 부분이 의아하긴 하지만 앞에서 언급한 대로 박스안에 물체가 있을 확률 정도로 해석하면 될거 같다. Anchor를 만드는 방법은 scale 과 aspect ratio라는 것을 이용해서 가능하다. 여기선 3개의 scale을 사용하였고 3개의 aspect ratio를 사용하였다. 따라서 서로 다른 총 9개의 Anchor들을 만들어 낼 수 있었다. 

Multi-scale detection를 수행하는 유명한 방법으로는 두 가지가 존재한다. 첫 번째는 입력 이미지를 여러개의 스케일로 나누어서 각각을 계산하는 것이다. 이는 여러개의 스케일에서 특징을 추출 할 수 있다는 장점이 있어서 종종 유용할 지도 모르나 스케일 마다 계산을 진행해야 하기 때문에 time-consuming 관점에서 별로 좋지 못하다. 두 번째는 입력이미지의 스케일을 변화시켜서 하는 것이 아니라 적용되는 filter의 스케일을 변화시켜 특징들을 추출하는 것이다. 하지만 여러 scale의 feature map을 생성할 경우 selective search보다 훨씬 더 많은 proposal들을 추출할지도 모르기 때문에 더욱 오래걸리고 False Positive한 결과를 도출한다. 그래서 주장하는 것이 간단한 CNN bbox regressor를 selective search를 대신하여 사용하는 것이다. 이것이 아마 RPN 인 것 같다. 이는 더욱 적절한 region proposal들을 추출하는 것을 도와준다.

논문처럼 Anchor가 9개이면 한번의 convolution 연산을 사용할 때 마다 9개의 proposals가 생기는 샘인데 이는 두 번쨰 방법의 단점과 마찬가지로 너무 많은 region proposal 들을 생기게 한다. 그래서 우리는 region proposal들을 줄이기 위해 binary classifier를 사용한다. 이 분류기에서는 아마 배경을 많이 포함하는 정도에 따라서 분류하는데 생각해보면 배경을 더 많이 포함한다는 것은 그 region proposal에 객체가 존재할 확률이 작아진다는 이야기이고 따라서 배경을 많이 포함하는 region은 삭제되는 것 같다. 따라서 두 번재 방법의 문제를 해결할 수 있다는 것이다.

## Region Proposal Networks(RPN)

단순히 어떤 class의 객체 인지는 파악이 안되지만 그 bbox안에 객체가 있는지를 나타내는 값으로 이해하면 될 거 같다. 수 많은 anchor box들에 한해서 objectness score와 bounding box coefficient를 구하는 역할을 한다. 여기서 Objectness는 논문에 다음과 같이 부연설명되어 있다. 

> “Objectness” measures membership to a set of object classes vs background.

![RPN](./img/RPN.png)

## Training 
















