# elerea

elerea에 관한 자료 공부. 우선 해당 라이브러리의 기반이 된 논문인 [Efficient and Compositional Higher-Order Streams](http://sgate.emt.bme.hu/documents/patai/publications/PataiWFLP2010.pdf)에 대한 내용을 먼저 공부하고 정리한 다음 실제 구현에 대해서 차근 차근 뜯어볼 것.

## Efficient and Compositional Higher-Order Streams

일단 논문을 읽으면서 논문의 순서를 거의 그대로 따라 내가 이해한 내용을 적는다. 내용 정리는 제대로 이해하고 다시 한 번 하자.

### 도입

순수 함수형 프로그래밍의 참조 투명성(referential transparency)은 굉장히 강력한 특성이다. 하지만 순수 함수만 가지고 상호작용성이 강한 프로그램을 작성하기는 굉장히 힘들다는 문제가 있다. 이 문제를 해결하기 위해 스트림 기반 프로그래밍(stream based programming)이 등장. 이 방식의 기본적인 아이디어는 프로그램에서 등장하는 모든 변수가 **각 시간에 따른 모든 값**을 나타내는 것으로 생각하는 것이다. 예를 들어서 `x+y`와 같은 표현식은 단순히 x,y라는 두 개의 값의 합을 말하는 것이 아니라 `x`라는 스트림과 `y`라는 스트림의 값을 합한 새로운 스트림으로 생각하는 것이다. 실제로 x,y라는 값은 해당 값을 참조하는 시점에 따라 각각 다른 값을 가지게 되겠지만 그 값들을 마치 순수한 값인 것처럼 쓸 수 있게 되는 것이다. functional reactive programming은 이 방식을 좀 더 확장해서 몇 가지 higher-order 구조를 덧붙인 것. 제일 처음 이것을 제시한 것은 [Fran](http://conal.net/papers/icfp97/)이다. 여기서 time-varying value를 first-class 개체로 다루는 방식을 제시함. 이 논문을 보면 각 스트림의 시작 시점을 어떻게 다뤄야할 지가 중요하다는 것을 알 수 있다. 모든 스트림이 글로벌 타임(프로그램의 실행 시작 시점을 기준으로 생각하는 것)을 따르게 할 수도 있고, 로컬 타임(모든 스트림이 그 스트림이 생성되는 시점으로부터 시간을 계산하는 것)을 따르게 할 수도 있다. 글로벌 타임을 따르게 할 경우 다루기 편하지만 시간/공간의 소모가 크다. 모든 값들이 프로그램 시작 시점부터의 값 변화를 저장하고 있어야만 하기 때문이다. 반면 로컬 타임을 따르는 경우 이런 문제를 겪지는 않지만 참조 투명성이 깨지기 쉬운 문제가 있다. `x+y`의 의미가 x 혹은 y 값의 시점 변경에 따라 달라지기 때문이다.

이 논문에서는 참조 투명성과 효율성을 같이 가져가는 접근법을 다룸.


### Higher-Order 스트림의 문제

스트림은 `head`와 `tail`을 통해 관찰 가능한 오브젝트다. 원소 타입이 `a`일 때 스트림은 `Stream a`타입으로 나타낼 수 있음. 스트림은 applicative functor이고, 따라서 `pure`와 `ap` 함수를 정의할 수 있다. 임의의 정적 네트워크를 구성하기 위해서 시작 시점으로부터의 단위 시간 딜레이 함수가 필요. 피드백은 `value recursion`으로 표현 가능.

#### First-order stream constructor

```Haskell
cons x s = < x s_0 s_1 s_2 s_3 ... >

head (cons x s) is equal to x
tail (cons x s) is equal to s

pure x = < x x x x x ... >

head (pure x) is equal to x
tail (pure x) is equal to pure x

f `ap` x = < (f_0 x_0) (f_1 x_1) (f_2) (x_2) (f_3 x_3) ... >

head (f `ap` x) is equal to (head f) (head x)
tail (f `ap` x) is equal to (tail f) `ap` (tail x)
```

#### dynamic network

동적 네트워크를 구성하는 경우를 생각해보자. 만약 higher-order 스트림을 펼치는(flatten) 연산을 정의할 수 있다면 higher-order 스트림을 실제 동적 네트워크로 바꿀 수 있다(어째서? - 잘 모르겠음. 좀 더 봐야 알 듯. 적혀 있는 설명만으로는 이해를 못하겠음). 이 건 모나드의 `join` 연산을 따라야 함. `join` 함수가 뭔 일을 하는 지 생각하면 이건 직관적인 듯. 

```Haskell
head (join s) is equal to head (head s)
tail (join s) is equal to join (fmap tail (tail s))
```

`join s`는 스트림의 스트림 s에서 대각선 성분 `s_00 s_11 s_22 s_33 ...`을 나타낸다고 생각하면 됨. 하지만 여기서 이게 실용적이지 못한 점은, 효율성이 굉장히 떨어지기 때문이다. n번의 샘플링은 n^2번의 스텝을 필요로 하니까. 

### Doing without Cons

