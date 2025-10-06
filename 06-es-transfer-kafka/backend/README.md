
### How to READ

Saga 는 의미적 잠금이다.    
여러 비즈니스 트랜잭션 (입금 + 출금) 이 전부 끝나야 endSaga() 를 호출할 수 있다.   
혹은 중간에 cancel 해도 endSaga 로 호출.

```
TransferSagaCoordinator.on(TransferCreated)
-> TransferSaga 
-> TransferSaga.on(SagaBegan) -> startSaga -> EventSourcedSaga.inSaga = true

@Scheduled(fixedDelay = 100) SagaMessageRelay.publish -> sagaEventStore.retrieveUnrelayedEvents

SagaEventStore.retrieveUnrelayedEvents() -> SagaEventRepository.findAllByRelayed(false)

SagaEventStore.update(Event) -> eventJpo.setRelayed(true)


TransferSagaCoordinator.on(Withdrawed)
-> saga = sagaStore.load(transferId) 
-> TransferSaga.inSaga()? TransferSaga.withdraw(); sagaStore.save(saga)

TransferSaga.withdraw() -> EventSourcedSaga.apply(SagaWithdrawed) 
-> TransferSaga.on(SagaWithdrawed) -> withdrawed = true; if(deposited && withdrawed){ endSaga() }

```

`TransferSagaCoordinator` 에서는 `EventListener` 만 존재하며, 
```
TransferCreated, Withdrawed, Deposited, WithdrawFailed,
SagaWithdrawed, SagaDeposited, SagaCompleted, SagaTimeExpired, SagaCanceled
```
이벤트를 처리한다. 방식은 다음과 같은데,     
TransferSaga 생성 혹은 로드
생성, 로드 하는 부분에서, EventSourcedSaga.apply 를 호출하여, 
sagaStore.save() 를 호출. 내부적으로 EventSourcedSaga 를 분리하여, SagaJpo 와 SagaEventJpo 로 저장.   
SagaEventJpo 는 EventSourcedSaga 내부의 events 리스트를 하나하나 꺼내어서 변환.  
EventSourcedSaga 의 event 들은 TransferSaga 의  
```
TransferSaga(BeginSaga) -> apply(SagaBegan)
depoist() -> apply(SagaDeposited)
withdraw() -> apply(SagaWithdrawed)
complete() -> apply(SagaCompleted)
cancel() -> apply(SagaCanceled)
```
함수를 호출될때 넣어짐. 그리고 sequence 가 존재하며, 해당 SagaXXX 이벤트가 몇번째로 발생했는지 알 수 있음.       
List 구조이기때문에, 해당 sagaId (=transferId) 에 발행된 모든 Saga Event 를 저장 및 로드함.     
SagaStore 의 save, load 함수 참고.    


SagaTimeout 의 경우에는 sagaId 가 correlationId 이다. 왜 이건 transferId 가 아닌거지?   




```sql
INSERT INTO TB_SAGA (ID, IN_SAGA, SEQUENCE, TYPE, VERSION) VALUES ('b997b8ab', false, 0, 'io.cosmos.transfer.saga.transfer.TransferSaga', 1);
INSERT INTO TB_SAGA_EVENT (ID, PAYLOAD, RELAYED, SAGA_ID, SEQUENCE, TIME, TYPE) VALUES ('16a8394d-6733-41c2-9754-9ea702bc4773', '{"transferId":"b997b8ab","fromAccountNo":"8fce365a","toAccountNo":"5ded8b57","amount":600}', true, 'b997b8ab', 1, 1659846465581, 'io.cosmos.transfer.saga.transfer.event.TransferSagaBegan');
INSERT INTO TB_SAGA_TIMEOUT (CORRELATION_ID, EXPIRE_TIME, SAGA_TYPE, COMPLETE) VALUES ('b997b8ab', '2022-08-07 13:23:13.119', 'Transfer', false);

```