---
title: 따닥 버튼을 눌러서 생기는 동시성 문제 해결기
date: YYYY-MM-DD HH:MM:SS +09:00
categories: [백엔드, 안나]
tags:
  [
    동시성, 따닥 문제, 락, 트랜잭션 격리 수준
  ]
---
# 😲문제 상황

우리 서비스는 참여 시 유저가 몇 번 요청하든 한 번의 참여만 가능하도록 설계되어있어요.

![image](https://github.com/user-attachments/assets/05476479-4dd2-4394-8eeb-5618a26145ec)

하지만 QA 단계에서 `따닥` 하고 버튼을 눌러 여러 번 참여를 시도했을 때 위와 같이 같은 회원이 두 번 이상 참여되는 것을 확인할 수 있습니다.

현재 참여 인원은 최대 모집 인원을 초과할 수 없도록 서버의 로직을 작성하였음에도 불구하고요.

이러한 예외 상황까지 핸들링하는 것이 저희가 `한 회원 당 한 번의 참여만 가능하게 한다.` 라고 세운 요구사항을 충족시키는 조건이 되겠죠.

# 🤔문제 분석

그렇다면 이 문제는 왜 생길까요?

모임 참여 시 한 회원이 두 번 요청을 했을 때 정상적인 플로우는 다음과 같습니다.

![image](https://github.com/user-attachments/assets/dbbea420-99df-4104-864b-c38b5c33bb60)

```java
1. 처음 참여를 시도한다.
2. 현재 참여 인원과 최대 참여 인원을 비교한다. 이때 회원이 참여 중인지 확인한다. (SELECT)
3. 최대 참여 인원을 넘지 않는 다면 모임에 참여한다. (INSERT)
5. 두 번째 참여를 시도한다. 
6. 최대 참여 인원을 넘게 되어 모임에 참여할 수 없다. 
```

하지만 여러 요청이 동시에 생기면 어떻게 될까요? 아래 상황을 가정해봅시다.

![image](https://github.com/user-attachments/assets/9464568c-1e93-4dbe-9f04-17082b76ce7e)

첫 번째 요청에 대한 INSERT 쿼리가 날라가기 전에 두 번째 참여 요청이 SELECT 쿼리로 **이미 참여하였는 지 확인**을 ****시도하게 되면, 참여하지 않았으니 INSERT가 통과 될 거에요. 그러면 데이터베이스에는 똑같은 회원이 두 번 참여된 상태로 저장될 것입니다.

정리하자면, 우리가 해결해야 할 핵심은 첫 번째 요청에 대한 INSERT가 수행된 다음 두 번째 요청에 대한 SELECT가 수행되는 것입니다.

![image](https://github.com/user-attachments/assets/256dc1f6-a576-4349-8db4-c00476a796c1)

![image](https://github.com/user-attachments/assets/f556bfeb-d4b3-4135-a2a8-55ec9b8fa6bd)

실제로 동시성 테스트를 해보았을 때 같은 회원이 어김없이 모임에 여러 번 참여할 수 있는 것을 확인하였습니다.

# 해결 방법은 뭐가 있을까?

어떻게 첫 번째 트랜잭션에 대한 INSERT가 끝났을 때 두 번째 SELECT가 수행되는 것을 보장할 수 있을까요?

## 트랜잭션 격리수준

InnoDB 스토리지 엔진을 사용하는 MySQL의 기본 트랜잭션 격리 수준은 REPEATABLE READ입니다.

REPEATABLE READ 격리 수준에서는 INSERT를 위한 트랜잭션이 시작되었을 때 다른 트랜잭션이 해당 레코드에 대해 SELECT를 시도하면, 언두 로그에 저장된 레코드를 읽게 됩니다.

아래 그림을 보면 트랜잭션 B에서 똑같이 별을 조회했는데 트랜잭션 A의 커밋 여부에 따라 다른 결과가 나오죠.

![image](https://github.com/user-attachments/assets/a34a915a-c4e9-4a81-b8ea-b3d86354bdf7)

이를 PANTHOM READ문제라고 불러요. SELECT 쿼리에 대한 결과가, 다른 트랜잭션의 INSERT 여부에 영향을 받게 됩니다.

이 문제를 해결하기 위해서는 트랜잭션 격리 수준을 올리는 방법이 있습니다. 격리 수준을 `SERIALIZABLE`로 변경하면, 한 번에 오직 하나의 트랜잭션만 한 번에 실행되기 때문에 서로 침범할 일이 없어요.

그래서 참여를 두 번 눌렀을 때 첫 번째 참여에 대한 트랜잭션이 끝나야 그제야 두 번째 참여가 시작할 수 있습니다. 그러면 우리가 설계한 비즈니스 로직도 무리 없이 실행될거에요.

두 번 이상 동시에 참여 버튼을 눌러도 한 번만 참여가 되는 것을 테스트하기 위해 테스트 검증 로직을 변경하였습니다.

![image](https://github.com/user-attachments/assets/edd83d73-bb9f-411b-9f29-bc296f9937e6)

![image](https://github.com/user-attachments/assets/cbe6cecd-d599-40a3-a8d1-273f5e3b9260)

트랜잭션 격리 수준을 SERIALIZABLE로 하니까 따닥 문제가 해결됨을 확인하였습니다.

하지만 `SERIALIZABLE`은 레코드도, 테이블 단위도 아닌 데이터베이스 단위로 적용되기 때문에 데이터베이스 단의 모든 쿼리가 단 하나만 동시에 일어날 수 있어요. **성능 저하가 클 것**으로 예상됩니다.

얼마나 성능이 저하될 지 확인해봅시다. 1000명의 사용자가 한 모임에 동시에 참여 요청을 보내는 상황을 가정해볼게요.

기획 상 한 집단 (회사, 학교 등 .. )에서 하나의 모임에 참여하기 때문에 가능한 동시 접속 사용자 수를 1000명 정도로 예상하였습니다.

```cpp
	@DisplayName("다른 회원이 동시에 여러 번 참여 요청을 하면 모두 참여 가능하다")
	@Test
	void chamyoMoimConcurrently() throws InterruptedException {
		int threadCount = 1000;
		ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
		CountDownLatch latch = new CountDownLatch(threadCount);

		Darakbang darakbang = DarakbangFixture.getDarakbangWithMouda();
		darakbangRepository.save(darakbang);

		Moim moim = MoimFixture.getBasketballMoim(darakbang.getId());
		moimRepository.save(moim);

		long startTime = System.currentTimeMillis();
		for (int i = 0; i < threadCount; i++) {
			executorService.execute(() -> {
				try {
					Member member = MemberFixture.getAnna();
					memberRepository.save(member);

					DarakbangMember darakbangMember = DarakbangMemberFixture
						.getDarakbangMemberWithWooteco(darakbang, member);
					darakbangMemberRepository.save(darakbangMember);

					chamyoService.chamyoMoim(darakbang.getId(), moim.getId(), darakbangMember);
				} finally {
					latch.countDown();
				}
			});
		}
		long endTime = System.currentTimeMillis();
		System.out.printf("Test time : %d ms", endTime - startTime);
		latch.await();
		executorService.shutdown();

		assertThat(chamyoService.findAllChamyo(darakbang.getId(), moim.getId()).chamyos()).hasSize(1000);
	}
```

SERIALIZABLE 격리수준으로 테스트했을 때 데드락 등의 오류가 많이 생겨 1000건 중 21건만이 정상적으로 INSERT 되었습니다. 총 걸린 시간은 11,111ms로, 건 당 529ms 정도입니다.

반대로 REPEATABLE READ 격리수준으로 테스트했을 때 1000건 중 646건이 저장되었어요. 건 당 68ms 정도입니다.

정리하자면, SERIALIZABLE을 사용하는 것으로 바꾼다면 쿼리 성능이 680% 로 저하되는 것을 확인하였어요.

## 락 (Lock)

이번에는 락을 사용해서 문제를 해결해봅시다.

한 명이 동시에 하나의 모임에 조회할 때, 이 모임이 같은 모임 객체임을 활용해 레코드 락을 사용하였습니다.

레코드 락은 테이블 단위가 아닌 레코드 하나에 대해 다른 트랜잭션이 접근할 수 없도록 락을 거는 겁니다.

![image](https://github.com/user-attachments/assets/e38ab482-35e2-4c6c-985c-a5b16c001077)

참여하는 비즈니스 로직에는 모임 정보를 조회하는 부분이 있어요. 이 모임 객체를 조회할 때 락을 걸면 다른 트랜잭션에서 동시에 모임을 조회하지 못하고 대기 상태가 될 것입니다. 따라서 이후 `validateCanChamyoMoim` 메서드에서 **이미 참여한 회원인지 조회할 때 앞선 트랜잭션의 INSERT 결과가 반영**되어있을 것이다.

```java
	@Lock(LockModeType.PESSIMISTIC_WRITE)
	@Query("select m from Moim m where m.id = :moimId")
	Optional<Moim> findByIdForUpdate(@Param("moimId") Long moimId);
```

배타적 쓰기 락을 걸어서 참여하는 모임 레코드에 다른 트랜잭션의 읽기와 쓰기를 모두 불가능하게 만들었습니다.

이제 동시성을 테스트하는 코드를 돌리면 제일 처음 실행된 참여 요청만 저장될 것 입니다. 두 번째 요청은 **이미 참여하였어요!** 라는 오류 메시지를 터뜨리며 실행되지 않을 거에요.

![image](https://github.com/user-attachments/assets/3c825e23-981b-4543-bbbe-08586c3f8cc7)

![image](https://github.com/user-attachments/assets/d44f8f39-19b9-4e25-96ee-08b09a3e66e6)

락을 사용했을 때 발생할 수 있는 문제에 대해서 생각해봤어요.

비관적 / 배타적 락을 사용하면 특정 모임 레코드에 읽기와 쓰기가 불가능합니다. 즉 모임 참여를 시도하는 트랜잭션이 진행 중일 때, 다른 트랜잭션은 이 모임 데이터를 조회할 수 없어요.

하지만 (1) 테이블이나 데이터베이스 단위가 아닌 특정 레코드에 대해서만 트랜잭션 대기를 걸 수 있는 점 (2) 참여와 모임 조회가 동시에 일어났을 때 *모임 조회가 실패하는 리스크*보다 *연속 참여를 하는 리스크*가 큰 점을 고려해서 채택하였습니다.

이에 대한 다양한 방법에 대해서 같이 논의해보고 싶네요! 😀
