# KIT Transformer
## 3.1 인코더/디코더 스택
**인코더:** 인코더 스택은 동일한 6개의 인코더가 쌓여 구성됩니다. 
각 인코더엔 두 개의 하위 층이 존재합니다. 
첫 번째 하위층은 멀티 헤드 셀프 어텐션층이며 두 번째 하위층은 포지션 와이즈 완전 연결 피드 포워드 신경망입니다. 
두 개의 하위층 다음엔 잔차 연결층[11]과 정규화층[1]이 존재합니다.
따라서 각 서브층의 출력은 ```LayerNorm(x + Sublayer(x))``` 이며 Sublayer(x)는 하위층에서 구현되는 함수입니다.
이러한 잔차 연결을 용이하게 하기 위해 모든 트랜스포머의 모든 하위층과 임베딩층은 d_model인 512 차원의 출력을 생성합니다.

**디코더:** 디코더 스택은 동일한 6개의 디코더가 쌓여 구성됩니다. 
인코더에 존재하는 두 개의 하위층 외에 디코더는 인코더 스택의 출력에 대해 멀티 헤드 어텐션을 수행하는 세 번째 하위층이 추가됩니다.
인코더와 동일하게 각 하위층 다음엔 잔차 연결층과 정규화층이 있습니다.
또한 현재 위치가 다음 위치로 이동하는 것을 방지하기 위해 디코더 스택 내부의 셀프 어텐션 하위층을 수정하였습니다.
출력 임베딩의 각 원소는 하나의 위치만을 나타내므로 마스킹을 통해 i 위치의 예측은 i 이전의 원소들에게만 영향받을 수 있음을 보장합니다.

## 3.2 어텐션
어텐션 함수는 쿼리와 키-값 쌍(쿼리, 키, 값은 모두 벡터)을 출력에 대응시키는 함수입니다. 출력은 값들의 가중합으로 계산되며 각 값에 할당된 가중치들은 쿼리와 대응되는 키의 호환 함수에 의해 계산됩니다.

### 3.2.1 스케일드 닷 프로덕트 어텐션
트랜스포머에서 적용되는 특별한 어텐션을 "스케일드 닷 프로덕트 어텐션"이라고 합니다. 
입력은 d_k 차원인 쿼리 및 키와 d_v 차원인 값으로 구성됩니다. 
쿼리는 모든 키에 대해 내적을 계산한 후 √d_k로 나누고 값에 대한 가중치를 얻기 위해 소프트맥스 함수를 적용시킵니다.

실제로는 쿼리들을 합쳐 Q 행렬을 만든 후 행렬 연산을 통해 어텐션 함수를 동시에 적용시킵니다. 
키와 값 역시 동일한 원리로 K 행렬과 V 행렬로 만듭니다.
이후 만들어진 행렬을 아래의 식에 대입하여 출력을 얻습니다.
```
Attention(Q, K, V) = softmax(QK^T / √d_k)V
```

어텐션 함수로 쓰이는 가장 일반적인 함수는 가산 어텐션[2]과 닷 프로덕트(내적) 어텐션입니다. 
닷 프로덕트 어텐션은 √d_k로 나눈다는 과정이 없다는 것을 제외하면 완벽히 동일합니다.
가산 어텐션은 단일 은닉층으로 구성된 순방향 네트워크를 사용하여 호환성 함수를 계산합니다. 
가산 어텐션과 닷 프로덕트 어텐션은 이론적인 복잡도는 비슷하지만 닷 프로덕트 어텐션 쪽이 조금 더 빠르고 공간복잡도 상으로도 효율적인데, 이는 최적화된 행렬 곱셈 코드를 이용하여 구현할 수 있기 때문입니다.

하지만 d_k가 작을 경우 두 어텐션이 비슷하게 수행되는 반면 d_k의 값이 클 경우 닷 프로덕트 어텐션을 스케일링 없이 사용한다면 가산 어텐션 쪽이 더 높은 성능을 나타내게 됩니다.[3] 
이는 d_k의 값이 커지게 되면서 내적을 하는 동안 크기가 커지게 되고 이것이 softmax 함수에까지 영향을 미친 것으로 추정합니다. 
이러한 현상을 방지하기 위해 트랜스포머에서는 내적을 하는 중 √d_k로 나눠 스케일링을 수행합니다.

### 3.2.2 멀티-헤드 어텐션
d_model 차원의 키, 값, 쿼리들로 단일 어텐션 함수를 수행하는 것보다 d_k와 d_v 차원으로 각각 h번만큼 선형적으로 투영하는 것이 더 낫다는 사실을 발견했습니다. 
이런 식으로 투영된 쿼리, 키, 값들 각각에 어텐션 함수를 병렬적으로 수행하여 d_v 차원의 출력 값을 산출합니다. 
이들은 연결되어 다시 한 번 더 투영되어 그림2와 같은 최종적인 값이 됩니다.

멀티-헤드 어텐션은 다른 위치에서 표현되는 정보를 수집할 수 있게합니다. 단일 어텐션의 경우 평균화 과정이 이를 방지합니다.

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h)W^O
이때, head_i = Attention(QW^Q, KW^K, VW^V)
```

트랜스포머에서는 병렬 어텐션층(헤드)를 총 8개 두었습니다. 
이 경우 d_k = d_v = d_model/h = 64입니다. 
각 헤드의 벡터는 차원이 감소한 상태이므로 전체 계산 비용 자체는 원본 벡터를 단일 어텐션 함수에 적용시킨 것과 비슷합니다.

### 3.2.3 트랜스포머의 어텐션 활용
트랜스포머는 아래와 같은 방법들로 멀티-헤드 어텐션을 사용합니다.
* 인코더-디코더 어텐션층에서 쿼리로 이전 디코더층을, 키와 값으로는 인코더의 출력을 사용합니다. 
  이로 인해 디코더의 모든 단어들이 입력 문장의 모든 단어들에 영향을 끼칠 수 있습니다. 
  또한 [38, 9, 2] 와 같은 시퀀스 투 시퀀스 모델에서의 어텐션 매커니즘과 유사한 효과를 낼 수 있습니다.
* 인코더엔 셀프 어텐션층이 있습니다.
셀프 어텐션층에서는 모든 키, 값, 쿼리의 출처가 동일합니다. 인코더 내부 셀프 어텐션의 경우 출처는 이전 인코더층입니다.
이를 통해 인코더 내부의 각각의 단어는 이전 인코더층의 모든 단어들에 영향을 끼칠 수 있습니다.
* 디코더에도 인코더와 유사하게 셀프 어텐션층이 있어 디코더 내부 각각의 단어들이 이전 디코더의 모든 단어에 영향을 끼칠 수 있도록 합니다.
디코더에서는 미래의 단어가 현재의 단어에게 영향을 끼칠 수 없게하여 자동 회귀하는 특징을 유지할 필요가 있습니다.
트랜스포머에서는 이러한 잘못된 연결을 스케일드 닷-프로덕트 어텐션 수행 중 마스킹 처리하도록 하여 소프트맥스 함수를 거쳤을 때 결과적으로 아무런 영향을 주지 못하게 만들었습니다. 그림 2를 참고하세요.

## 3.3 포지션-와이즈 피드 포워드 신경망
트랜스포머의 인코더와 디코더엔 어텐션층 뿐만 아니라 피드 포워드 완전연결층을 포함하며 각 위치에서 개별적이지만 동일하게 동작합니다. 
피드 포워드 완전연결층은 두 개의 선형 변환으로 구성되며 ReLU 활성화 함수를 사용합니다.

```
FFN(x) = max(0, xW1 + b1)W2 + b2
```
선형 변환은 층 내부에서는 동일하게 수행되지만 층마다는 다른 파라미터를 사용합니다. 
이는 커널 크기가 1인 두 개의 컨볼루션과 비슷한 효과를 냅니다.
입력과 출력의 차원(d_model)은 512이며 피드 포워드 신경망의 차원(d_ff)은 2048입니다.


## 3.4 임베딩과 소프트맥스
다른 시퀀스 변환 모델과 유사하게 입력 토큰과 출력 토큰을 d_model 차원의 벡터로 변환시키기 위해 학습된 임베딩을 사용했습니다.
또한 디코더의 출력을 예측된 다음 토큰 확률로 변환시키기 위해 학습된 선형 변환과 소프트맥스 함수를 사용했습니다.
트랜스포머는 사전 소프트맥스 선형 변환과 두 개의 임베딩층 사이에 동일한 가중치 행렬을 공유합니다.
임베딩층에서는 해당 가중치를 √d_model과 곱합니다.

## 3.5 포지셔널 인코딩
트랜스포머는 회귀 혹은 컨볼루션을 사용하지 않기 때문에 모델이 시퀀스 순서 정보를 이용하려면 시퀀스 내부 토큰들의 상대적 혹은 절대적 위치 정보를 주입해줘야 합니다.
이를 위해 트랜스포머는 인코더와 디코더 스택에 입력되기 전 "포지셔널 인코딩" 벡터를 입력 임베딩 벡터와 더하는 과정을 거칩니다.
포지셔널 인코딩 벡터는 d_model 차원이므로 두 벡터를 더할 수 있습니다.
포지셔널 인코딩은 학습된 것과 고정된 것이 있습니다.[9]


트랜스포머에서는 다른 파형을 지닌 사인과 코사인 함수를 사용합니다.

```
PE(pos,2i) = sin(pos/10000**(2*i/dmodel))
PE(pos,2i+1) = cos(pos/10000**(2i/dmodel))
```

위 식에서 pos는 행렬의 행 인덱스를 나타내고, i는 열 인덱스를 나타냅니다. 
즉 포지셔널 인코딩의 각 원소는 사인/코사인 함수의 값에 해당합니다. 
파장은 2π에서 10000 * 2π까지의 기하학적 파형을 형성합니다.
우리는 모델이 상대적인 위치로 인해 더 잘 학습할 것이라고 가정하였고 따라서 고정된 오프셋 k에 대해 PE_pos + k를 PE_pos의 선형 함수로 표현시킬 수 있는 사인과 코사인 함수를 채택했습니다.

또한 학습된 포지셔널 임베딩[9]을 대신 사용하여 실험한 결과 결과에 큰 차이가 없음을 밝혀냈습니다. (표3 E행 참조) 그럼에도 우리가 사인과 코사인 함수를 사용한 이유는 학습된 시퀀스 길이보다 더 긴 시퀀스에 대해 더욱 유연하게 대처할 수 있기 때문입니다.