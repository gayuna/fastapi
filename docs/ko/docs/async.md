# 동시성과 async / await

*경로 작동 함수*에서의 `async def` 문법에 대한 세부사항과 비동기 코드, 동시성 및 병렬성에 대한 배경

## 바쁘신 경우

<strong>요약</strong>

다음과 같이 `await`를 사용해 호출하는 제3의 라이브러리를 사용하는 경우:

```Python
results = await some_library()
```

다음처럼 *경로 작동 함수*를 `async def`를 사용해 선언하십시오:

```Python hl_lines="2"
@app.get('/')
async def read_results():
    results = await some_library()
    return results
```

/// note | 참고

`async def`로 생성된 함수 내부에서만 `await`를 사용할 수 있습니다.

///

---

데이터베이스, API, 파일시스템 등과 의사소통하는 제3의 라이브러리를 사용하고, 그것이 `await`를 지원하지 않는 경우(현재 거의 모든 데이터베이스 라이브러리가 그러합니다), *경로 작동 함수*를 일반적인 `def`를 사용해 선언하십시오:

```Python hl_lines="2"
@app.get('/')
def results():
    results = some_library()
    return results
```

---

만약 당신의 응용프로그램이 (어째서인지) 다른 무엇과 의사소통하고 그것이 응답하기를 기다릴 필요가 없다면 `async def`를 사용하십시오.

---

모르겠다면, 그냥 `def`를 사용하십시오.

---

**참고**: *경로 작동 함수*에서 필요한만큼 `def`와 `async def`를 혼용할 수 있고, 가장 알맞은 것을 선택해서 정의할 수 있습니다. FastAPI가 자체적으로 알맞은 작업을 수행할 것입니다.

어찌되었든, 상기 어떠한 경우라도, FastAPI는 여전히 비동기적으로 작동하고 매우 빠릅니다.

그러나 상기 작업을 수행함으로써 어느 정도의 성능 최적화가 가능합니다.

## 기술적 세부사항

최신 파이썬 버전은 `async`와 `await` 문법과 함께 **“코루틴”**이라고 하는 것을 사용하는 **“비동기 코드”**를 지원합니다.

아래 섹션들에서 해당 문장을 부분별로 살펴보겠습니다:

* **비동기 코드**
* **`async`와 `await`**
* **코루틴**

## 비동기 코드

비동기 코드란 언어 💬 가 코드의 어느 한 부분에서, 컴퓨터 / 프로그램🤖에게 *다른 무언가*가 어딘가에서 끝날 때까지 기다려야한다고 말하는 방식입니다. *다른 무언가*가 “느린-파일" 📝 이라고 불린다고 가정해봅시다.

따라서 “느린-파일” 📝이 끝날때까지 컴퓨터는 다른 작업을 수행할 수 있습니다.

그 다음 컴퓨터 / 프로그램 🤖 은 다시 기다리고 있기 때문에 기회가 있을 때마다 다시 돌아오거나, 혹은 당시에 수행해야하는 작업들이 완료될 때마다 다시 돌아옵니다. 그리고 그것 🤖 은 기다리고 있던 작업 중 어느 것이 이미 완료되었는지, 그것 🤖 이 해야하는 모든 작업을 수행하면서 확인합니다.

다음으로, 그것 🤖 은 완료할 첫번째 작업에 착수하고(우리의 "느린-파일" 📝 이라고 가정합시다) 그에 대해 수행해야하는 작업을 계속합니다.

"다른 무언가를 기다리는 것"은 일반적으로 비교적 "느린" (프로세서와 RAM 메모리 속도에 비해) <abbr title="Input and Output">I/O</abbr> 작업을 의미합니다. 예를 들면 다음의 것들을 기다리는 것입니다:

* 네트워크를 통해 클라이언트로부터 전송되는 데이터
* 네트워크를 통해 클라이언트가 수신할, 당신의 프로그램으로부터 전송되는 데이터
* 시스템이 읽고 프로그램에 전달할 디스크 내의 파일 내용
* 당신의 프로그램이 시스템에 전달하는, 디스크에 작성될 내용
* 원격 API 작업
* 완료될 데이터베이스 작업
* 결과를 반환하는 데이터베이스 쿼리
* 기타

수행 시간의 대부분이 <abbr title="Input and Output">I/O</abbr> 작업을 기다리는데에 소요되기 때문에, "I/O에 묶인" 작업이라고 불립니다.

이것은 "비동기"라고 불리는데 컴퓨터 / 프로그램이 작업 결과를 가지고 일을 수행할 수 있도록, 느린 작업에 "동기화"되어 아무것도 하지 않으면서 작업이 완료될 정확한 시점만을 기다릴 필요가 없기 때문입니다.

이 대신에, "비동기" 시스템에서는, 작업은 일단 완료되면, 컴퓨터 / 프로그램이 수행하고 있는 일을 완료하고 이후 다시 돌아와서 그것의 결과를 받아 이를 사용해 작업을 지속할 때까지 잠시 (몇 마이크로초) 대기할 수 있습니다.

"동기"("비동기"의 반대)는 컴퓨터 / 프로그램이 상이한 작업들간 전환을 하기 전에 그것이 대기를 동반하게 될지라도 모든 순서를 따르기 때문에 "순차"라는 용어로도 흔히 불립니다.

### 동시성과 버거

위에서 설명한 **비동기** 코드에 대한 개념은 종종 **"동시성"**이라고도 불립니다. 이것은 **"병렬성"**과는 다릅니다.

**동시성**과 **병렬성**은 모두 "동시에 일어나는 서로 다른 일들"과 관련이 있습니다.

하지만 *동시성*과 *병렬성*의 세부적인 개념에는 꽤 차이가 있습니다.

차이를 확인하기 위해, 다음의 버거에 대한 이야기를 상상해보십시오:

### 동시 버거

당신은 짝사랑 상대 😍 와 패스트푸드 🍔 를 먹으러 갔습니다. 당신은 점원 💁 이 당신 앞에 있는 사람들의 주문을 받을 동안 줄을 서서 기다리고 있습니다.

이제 당신의 순서가 되어서, 당신은 당신과 짝사랑 상대 😍 를 위한 두 개의 고급스러운 버거 🍔 를 주문합니다.

당신이 돈을 냅니다 💸.

점원 💁 은 주방 👨‍🍳 에 요리를 하라고 전달하고, 따라서 그들은 당신의 버거 🍔 를 준비해야한다는 사실을 알게됩니다(그들이 지금은 당신 앞 고객들의 주문을 준비하고 있을지라도 말입니다).

점원 💁 은 당신의 순서가 적힌 번호표를 줍니다.

기다리는 동안, 당신은 짝사랑 상대 😍 와 함께 테이블을 고르고, 자리에 앉아 오랫동안 (당신이 주문한 버거는 꽤나 고급스럽기 때문에 준비하는데 시간이 조금 걸립니다 ✨🍔✨) 대화를 나눕니다.

짝사랑 상대 😍 와 테이블에 앉아서 버거 🍔 를 기다리는 동안, 그 사람 😍 이 얼마나 멋지고, 사랑스럽고, 똑똑한지 감탄하며 시간을 보냅니다 ✨😍✨.

짝사랑 상대 😍 와 기다리면서 얘기하는 동안, 때때로, 당신은 당신의 차례가 되었는지 보기 위해 카운터의 번호를 확인합니다.

그러다 어느 순간, 당신의 차례가 됩니다. 카운터에 가서, 버거 🍔 를 받고, 테이블로 다시 돌아옵니다.

당신과 짝사랑 상대 😍 는 버거 🍔 를 먹으며 좋은 시간을 보냅니다 ✨.

---

당신이 이 이야기에서 컴퓨터 / 프로그램 🤖 이라고 상상해보십시오.

줄을 서서 기다리는 동안, 당신은 아무것도 하지 않고 😴 당신의 차례를 기다리며, 어떠한 "생산적인" 일도 하지 않습니다. 하지만 점원 💁 이 (음식을 준비하지는 않고) 주문을 받기만 하기 때문에 줄이 빨리 줄어들어서 괜찮습니다.

그다음, 당신이 차례가 오면, 당신은 실제로 "생산적인" 일 🤓 을 합니다. 당신은 메뉴를 보고, 무엇을 먹을지 결정하고, 짝사랑 상대 😍 의 선택을 묻고, 돈을 내고 💸 , 맞는 카드를 냈는지 확인하고, 비용이 제대로 지불되었는지 확인하고, 주문이 제대로 들어갔는지 확인을 하는 작업 등등을 수행합니다.

하지만 이후에는, 버거 🍔 를 아직 받지 못했음에도, 버거가 준비될 때까지 기다려야 🕙 하기 때문에 점원 💁 과의 작업은 "일시정지" ⏸  상태입니다.

하지만 번호표를 받고 카운터에서 나와 테이블에 앉으면, 당신은 짝사랑 상대 😍 와 그 "작업"  ⏯ 🤓 에 번갈아가며 🔀  집중합니다. 그러면 당신은 다시 짝사랑 상대 😍 에게 작업을 거는 매우 "생산적인" 일 🤓 을 합니다.

점원 💁 이 카운터 화면에 당신의 번호를 표시함으로써 "버거 🍔 가 준비되었습니다"라고 해도, 당신은 즉시 뛰쳐나가지는 않을 것입니다. 당신은 당신의 번호를 갖고있고, 다른 사람들은 그들의 번호를 갖고있기 때문에, 아무도 당신의 버거 🍔 를 훔쳐가지 않는다는 사실을 알기 때문입니다.

그래서 당신은 짝사랑 상대 😍 가 이야기를 끝낼 때까지 기다린 후 (현재 작업 완료 ⏯ / 진행 중인 작업 처리 🤓 ), 정중하게 미소짓고 버거를 가지러 가겠다고 말합니다 ⏸.

그다음 당신은 카운터에 가서 🔀 , 초기 작업을 이제 완료하고 ⏯ , 버거 🍔 를 받고, 감사하다고 말하고 테이블로 가져옵니다. 이로써 카운터와의 상호작용 단계 / 작업이 종료됩니다 ⏹.

이전 작업인 "버거 받기"가 종료되면 ⏹ "버거 먹기"라는 새로운 작업이 생성됩니다 🔀 ⏯.

### 병렬 버거

이제 "동시 버거"가 아닌 "병렬 버거"를 상상해보십시오.

당신은 짝사랑 상대 😍 와 함께 병렬 패스트푸드 🍔 를 먹으러 갔습니다.

당신은 여러명(8명이라고 가정합니다)의 점원이 당신 앞 사람들의 주문을 받으며 동시에 요리 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳 도 하는 동안 줄을 서서 기다립니다.

당신 앞 모든 사람들이 버거가 준비될 때까지 카운터에서 떠나지 않고 기다립니다 🕙 . 왜냐하면 8명의 직원들이 다음 주문을 받기 전에 버거를 준비하러 가기 때문입니다.

마침내 당신의 차례가 왔고, 당신은 당신과 짝사랑 상대 😍 를 위한 두 개의 고급스러운 버거 🍔 를 주문합니다.

당신이 비용을 지불합니다 💸 .

점원이 주방에 갑니다 👨‍🍳 .

당신은 번호표가 없기 때문에 누구도 당신의 버거 🍔 를 대신 가져갈 수 없도록 카운터에 서서 기다립니다 🕙 .

당신과 짝사랑 상대 😍 는 다른 사람이 새치기해서 버거를 가져가지 못하게 하느라 바쁘기 때문에 🕙 , 짝사랑 상대에게 주의를 기울일 수 없습니다 😞 .

이것은 "동기" 작업이고, 당신은 점원/요리사 👨‍🍳 와 "동기화" 되었습니다. 당신은 기다리고 🕙 , 점원/요리사 👨‍🍳 가 버거 🍔  준비를 완료한 후 당신에게 주거나, 누군가가 그것을 가져가는 그 순간에 그 곳에 있어야합니다.

카운터 앞에서 오랫동안 기다린 후에 🕙 , 점원/요리사 👨‍🍳 가 당신의 버거 🍔 를 가지고 돌아옵니다.

당신은 버거를 받고 짝사랑 상대와 함께 테이블로 돌아옵니다.

단지 먹기만 하다가, 다 먹었습니다 🍔 ⏹.

카운터 앞에서 기다리면서 🕙  너무 많은 시간을 허비했기 때문에 대화를 하거나 작업을 걸 시간이 거의 없었습니다 😞 .

---

이 병렬 버거 시나리오에서, 당신은 기다리고 🕙 , 오랜 시간동안 "카운터에서 기다리는" 🕙 데에 주의를 기울이는 ⏯ 두 개의 프로세서(당신과 짝사랑 상대😍)를 가진 컴퓨터 / 프로그램 🤖 입니다.

패스트푸드점에는 8개의 프로세서(점원/요리사) 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳 가 있습니다. 동시 버거는 단 두 개(한 명의 직원과 한 명의 요리사) 💁 👨‍🍳 만을 가지고 있었습니다.

하지만 여전히, 병렬 버거 예시가 최선은 아닙니다 😞 .

---

이 예시는 버거🍔 이야기와 결이 같습니다.

더 "현실적인" 예시로, 은행을 상상해보십시오.

최근까지, 대다수의 은행에는 다수의 은행원들 👨‍💼👨‍💼👨‍💼👨‍💼 과 긴 줄 🕙🕙🕙🕙🕙🕙🕙🕙 이 있습니다.

모든 은행원들은 한 명 한 명의 고객들을 차례로 상대합니다 👨‍💼⏯ .

그리고 당신은 오랫동안 줄에서 기다려야하고 🕙 , 그렇지 않으면 당신의 차례를 잃게 됩니다.

아마 당신은 은행 🏦 심부름에 짝사랑 상대 😍 를 데려가고 싶지는 않을 것입니다.

### 버거 예시의 결론

"짝사랑 상대와의 패스트푸드점 버거" 시나리오에서, 오랜 기다림 🕙 이 있기 때문에 동시 시스템 ⏸🔀⏯ 을 사용하는 것이 더 합리적입니다.

대다수의 웹 응용프로그램의 경우가 그러합니다.

매우 많은 수의 유저가 있지만, 서버는 그들의 요청을 전송하기 위해 그닥-좋지-않은 연결을 기다려야 합니다 🕙 .

그리고 응답이 돌아올 때까지 다시 기다려야 합니다 🕙 .

이 "기다림" 🕙 은 마이크로초 단위이지만, 모두 더해지면, 결국에는 매우 긴 대기시간이 됩니다.

따라서 웹 API를 위해 비동기 ⏸🔀⏯ 코드를 사용하는 것이 합리적입니다.

대부분의 존재하는 유명한 파이썬 프레임워크 (Flask와 Django 등)은 새로운 비동기 기능들이 파이썬에 존재하기 전에 만들어졌습니다. 그래서, 그들의 배포 방식은 병렬 실행과 새로운 기능만큼 강력하지는 않은 예전 버전의 비동기 실행을 지원합니다.

비동기 웹 파이썬(ASGI)에 대한 주요 명세가 웹소켓을 지원하기 위해 Django에서 개발 되었음에도 그렇습니다.

이러한 종류의 비동기성은 (NodeJS는 병렬적이지 않음에도) NodeJS가 사랑받는 이유이고, 프로그래밍 언어로서의 Go의 강점입니다.

그리고 **FastAPI**를 사용함으로써 동일한 성능을 낼 수 있습니다.

또한 병렬성과 비동기성을 동시에 사용할 수 있기 때문에, 대부분의 테스트가 완료된 NodeJS 프레임워크보다 더 높은 성능을 얻고 C에 더 가까운 컴파일 언어인 Go와 동등한 성능을 얻을 수 있습니다<a href="https://www.techempower.com/benchmarks/#section=data-r17&hw=ph&test=query&l=zijmkf-1" class="external-link" target="_blank">(모두 Starlette 덕분입니다)</a>.

### 동시성이 병렬성보다 더 나은가?

그렇지 않습니다! 그것이 이야기의 교훈은 아닙니다.

동시성은 병렬성과 다릅니다. 그리고 그것은 많은 대기를 필요로하는 **특정한** 시나리오에서는 더 낫습니다. 이로 인해, 웹 응용프로그램 개발에서 동시성이 병렬성보다 일반적으로 훨씬 낫습니다. 하지만 모든 경우에 그런 것은 아닙니다.

따라서, 균형을 맞추기 위해, 다음의 짧은 이야기를 상상해보십시오:

> 당신은 크고, 더러운 집을 청소해야합니다.

*네, 이게 전부입니다*.

---

어디에도 대기 🕙 는 없고, 집안 곳곳에서 해야하는 많은 작업들만 있습니다.

버거 예시처럼 처음에는 거실, 그 다음은 부엌과 같은 식으로 순서를 정할 수도 있으나, 무엇도 기다리지 🕙 않고 계속해서 청소 작업만 수행하기 때문에, 순서는 아무런 영향을 미치지 않습니다.

순서가 있든 없든 동일한 시간이 소요될 것이고(동시성) 동일한 양의 작업을 하게 될 것입니다.

하지만 이 경우에서, 8명의 전(前)-점원/요리사이면서-현(現)-청소부 👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳👩‍🍳👨‍🍳 를 고용할 수 있고, 그들 각자(그리고 당신)가 집의 한 부분씩 맡아 청소를 한다면, 당신은 **병렬적**으로 작업을 수행할 수 있고, 조금의 도움이 있다면, 훨씬 더 빨리 끝낼 수 있습니다.

이 시나리오에서, (당신을 포함한) 각각의 청소부들은 프로세서가 될 것이고, 각자의 역할을 수행합니다.

실행 시간의 대부분이 대기가 아닌 실제 작업에 소요되고, 컴퓨터에서 작업은 <abbr title="Central Processing Unit">CPU</abbr>에서 이루어지므로, 이러한 문제를 "CPU에 묶였"다고 합니다.

---

CPU에 묶인 연산에 관한 흔한 예시는 복잡한 수학 처리를 필요로 하는 경우입니다.

예를 들어:

* **오디오** 또는 **이미지** 처리.
* **컴퓨터 비전**: 하나의 이미지는 수백개의 픽셀로 구성되어있고, 각 픽셀은 3개의 값 / 색을 갖고 있으며, 일반적으로 해당 픽셀들에 대해 동시에 무언가를 계산해야하는 처리.
* **머신러닝**: 일반적으로 많은 "행렬"과 "벡터" 곱셈이 필요합니다. 거대한 스프레드 시트에 수들이 있고 그 수들을 동시에 곱해야 한다고 생각해보십시오.
* **딥러닝**: 머신러닝의 하위영역으로, 동일한 예시가 적용됩니다. 단지 이 경우에는 하나의 스프레드 시트에 곱해야할 수들이 있는 것이 아니라, 거대한 세트의 스프레드 시트들이 있고, 많은 경우에, 이 모델들을 만들고 사용하기 위해 특수한 프로세서를 사용합니다.

### 동시성 + 병렬성: 웹 + 머신러닝

**FastAPI**를 사용하면 웹 개발에서는 매우 흔한 동시성의 이점을 (NodeJS의 주된 매력만큼) 얻을 수 있습니다.

뿐만 아니라 머신러닝 시스템과 같이 **CPU에 묶인** 작업을 위해 병렬성과 멀티프로세싱(다수의 프로세스를 병렬적으로 동작시키는 것)을 이용하는 것도 가능합니다.

파이썬이 **데이터 사이언스**, 머신러닝과 특히 딥러닝에 의 주된 언어라는 간단한 사실에 더해서, 이것은 FastAPI를 데이터 사이언스 / 머신러닝 웹 API와 응용프로그램에 (다른 것들보다) 좋은 선택지가 되게 합니다.

배포시 병렬을 어떻게 가능하게 하는지 알고싶다면, [배포](deployment/index.md){.internal-link target=_blank}문서를 참고하십시오.

## `async`와  `await`

최신 파이썬 버전에는 비동기 코드를 정의하는 매우 직관적인 방법이 있습니다. 이는 이것을 평범한 "순차적" 코드로 보이게 하고, 적절한 순간에 당신을 위해 "대기"합니다.

연산이 결과를 전달하기 전에 대기를 해야하고 새로운 파이썬 기능들을 지원한다면, 이렇게 코드를 작성할 수 있습니다:

```Python
burgers = await get_burgers(2)
```

여기서 핵심은 `await`입니다. 이것은 파이썬에게 `burgers` 결과를 저장하기 이전에 `get_burgers(2)`의 작업이 완료되기를 🕙 기다리라고 ⏸ 말합니다. 이로 인해, 파이썬은 그동안 (다른 요청을 받는 것과 같은) 다른 작업을 수행해도 된다는 것을 🔀 ⏯ 알게될 것입니다.

`await`가 동작하기 위해, 이것은 비동기를 지원하는 함수 내부에 있어야 합니다. 이를 위해서 함수를 `async def`를 사용해 정의하기만 하면 됩니다:

```Python hl_lines="1"
async def get_burgers(number: int):
    # Do some asynchronous stuff to create the burgers
    return burgers
```

...`def`를 사용하는 대신:

```Python hl_lines="2"
# This is not asynchronous
def get_sequential_burgers(number: int):
    # Do some sequential stuff to create the burgers
    return burgers
```

`async def`를 사용하면, 파이썬은 해당 함수 내에서 `await` 표현에 주의해야한다는 사실과, 해당 함수의 실행을 "일시정지"⏸하고 다시 돌아오기 전까지 다른 작업을 수행🔀할 수 있다는 것을 알게됩니다.

`async def` 함수를 호출하고자 할 때, "대기"해야합니다. 따라서, 아래는 동작하지 않습니다.

```Python
# This won't work, because get_burgers was defined with: async def
burgers = get_burgers(2)
```

---

따라서, `await`f를 사용해서 호출할 수 있는 라이브러리를 사용한다면, 다음과 같이 `async def`를 사용하는 *경로 작동 함수*를 생성해야 합니다:

```Python hl_lines="2-3"
@app.get('/burgers')
async def read_burgers():
    burgers = await get_burgers(2)
    return burgers
```

### 더 세부적인 기술적 사항

`await`가 `async def`를 사용하는 함수 내부에서만 사용이 가능하다는 것을 눈치채셨을 것입니다.

하지만 동시에, `async def`로 정의된 함수들은 "대기"되어야만 합니다. 따라서, `async def`를 사용한 함수들은 역시 `async def`를 사용한 함수 내부에서만 호출될 수 있습니다.

그렇다면 닭이 먼저냐, 달걀이 먼저냐, 첫 `async` 함수를 어떻게 호출할 수 있겠습니까?

**FastAPI**를 사용해 작업한다면 이것을 걱정하지 않아도 됩니다. 왜냐하면 그 "첫" 함수는 당신의 *경로 작동 함수*가 될 것이고, FastAPI는 어떻게 올바르게 처리할지 알고있기 때문입니다.

하지만 FastAPI를 사용하지 않고 `async` / `await`를 사용하고 싶다면, 이 역시 가능합니다.

### 당신만의 비동기 코드 작성하기

Starlette(그리고 FastAPI)는 <a href="https://anyio.readthedocs.io/en/stable/" class="external-link" target="_blank">AnyIO</a>를 기반으로 하고있고, 따라서 파이썬 표준 라이브러리인 <a href="https://docs.python.org/3/library/asyncio-task.html" class="external-link" target="_blank">asyncio</a> 및 <a href="https://trio.readthedocs.io/en/stable/" class="external-link" target="_blank">Trio</a>와 호환됩니다.

특히, 코드에서 고급 패턴이 필요한 고급 동시성을 사용하는 경우 직접적으로 <a href="https://anyio.readthedocs.io/en/stable/" class="external-link" target="_blank">AnyIO</a>를 사용할 수 있습니다.

FastAPI를 사용하지 않더라도, 높은 호환성 및 <a href="https://anyio.readthedocs.io/en/stable/" class="external-link" target="_blank">AnyIO</a>의 이점(예: *구조화된 동시성*)을 취하기 위해 <a href="https://anyio.readthedocs.io/en/stable/" class="external-link" target="_blank">AnyIO</a>를 사용해 비동기 응용프로그램을 작성할 수 있습니다.

### 비동기 코드의 다른 형태

파이썬에서 `async`와 `await`를 사용하게 된 것은 비교적 최근의 일입니다.

하지만 이로 인해 비동기 코드 작업이 훨씬 간단해졌습니다.

같은 (또는 거의 유사한) 문법은 최신 버전의 자바스크립트(브라우저와 NodeJS)에도 추가되었습니다.

하지만 그 이전에, 비동기 코드를 처리하는 것은 꽤 복잡하고 어려운 일이었습니다.

파이썬의 예전 버전이라면, 스레드 또는 <a href="https://www.gevent.org/" class="external-link" target="_blank">Gevent</a>를 사용할 수 있을 것입니다. 하지만 코드를 이해하고, 디버깅하고, 이에 대해 생각하는게 훨씬 복잡합니다.

예전 버전의 NodeJS / 브라우저 자바스크립트라면, "콜백 함수"를 사용했을 것입니다. 그리고 이로 인해 <a href="http://callbackhell.com/" class="external-link" target="_blank">콜백 지옥</a>에 빠지게 될 수 있습니다.

## 코루틴

**코루틴**은 `async def` 함수가 반환하는 것을 칭하는 매우 고급스러운 용어일 뿐입니다. 파이썬은 그것이 시작되고 어느 시점에서 완료되지만 내부에 `await`가 있을 때마다 내부적으로 일시정지⏸될 수도 있는 함수와 유사한 것이라는 사실을 알고있습니다.

그러나 `async` 및 `await`와 함께 비동기 코드를 사용하는 이 모든 기능들은 "코루틴"으로 간단히 요약됩니다. 이것은 Go의 주된 핵심 기능인 "고루틴"에 견줄 수 있습니다.

## 결론

상기 문장을 다시 한 번 봅시다:

> 최신 파이썬 버전은 **`async` 및 `await`** 문법과 함께 **“코루틴”**이라고 하는 것을 사용하는 **“비동기 코드”**를 지원합니다.

이제 이 말을 조금 더 이해할 수 있을 것입니다. ✨

이것이 (Starlette을 통해) FastAPI를 강하게 하면서 그것이 인상적인 성능을 낼 수 있게 합니다.

## 매우 세부적인 기술적 사항

/// warning | 경고

이 부분은 넘어가도 됩니다.

이것들은 **FastAPI**가 내부적으로 어떻게 동작하는지에 대한 매우 세부적인 기술사항입니다.

만약 기술적 지식(코루틴, 스레드, 블록킹 등)이 있고 FastAPI가 어떻게 `async def` vs `def`를 다루는지 궁금하다면, 계속하십시오.

///

### 경로 작동 함수

경로 작동 함수를 `async def` 대신 일반적인 `def`로 선언하는 경우, (서버를 차단하는 것처럼) 그것을 직접 호출하는 대신 대기중인 외부 스레드풀에서 실행됩니다.

만약 상기에 묘사된대로 동작하지 않는 비동기 프로그램을 사용해왔고 약간의 성능 향상 (약 100 나노초)을 위해 `def`를 사용해서 계산만을 위한 사소한 *경로 작동 함수*를 정의해왔다면, **FastAPI**는 이와는 반대라는 것에 주의하십시오. 이러한 경우에, *경로 작동 함수*가 블로킹 <abbr title="Input/Output: 디스크 읽기 또는 쓰기, 네트워크 통신.">I/O</abbr>를 수행하는 코드를 사용하지 않는 한 `async def`를 사용하는 편이 더 낫습니다.

하지만 두 경우 모두, FastAPI가 당신이 전에 사용하던 프레임워크보다 [더 빠를](index.md#_11){.internal-link target=_blank} (최소한 비견될) 확률이 높습니다.

### 의존성

의존성에도 동일하게 적용됩니다. 의존성이 `async def`가 아닌 표준 `def` 함수라면, 외부 스레드풀에서 실행됩니다.

### 하위-의존성

함수 정의시 매개변수로 서로를 필요로하는 다수의 의존성과 하위-의존성을 가질 수 있고, 그 중 일부는 `async def`로, 다른 일부는 일반적인 `def`로 생성되었을 수 있습니다. 이것은 여전히 잘 동작하고, 일반적인 `def`로 생성된 것들은 "대기"되는 대신에 (스레드풀로부터) 외부 스레드에서 호출됩니다.

### 다른 유틸리티 함수

직접 호출되는 다른 모든 유틸리티 함수는 일반적인 `def`나 `async def`로 생성될 수 있고 FastAPI는 이를 호출하는 방식에 영향을 미치지 않습니다.

이것은 FastAPI가 당신을 위해 호출하는 함수와는 반대입니다: *경로 작동 함수*와 의존성

만약 당신의 유틸리티 함수가 `def`를 사용한 일반적인 함수라면, 스레드풀에서가 아니라 직접 호출(당신이 코드에 작성한 대로)될 것이고, `async def`로 생성된 함수라면 코드에서 호출할 때 그 함수를 `await` 해야 합니다.

---

다시 말하지만, 이것은 당신이 이것에 대해 찾고있던 경우에 한해 유용할 매우 세부적인 기술사항입니다.

그렇지 않은 경우, 상기의 가이드라인만으로도 충분할 것입니다: [바쁘신 경우](#_1).
