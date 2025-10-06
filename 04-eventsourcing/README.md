
### How to READ

```
TransferEndpoint -> TransferService.transferMoney -> Transfer, aggregateStore.save(transfer)

Transfer -> EventSourcedAggregate.apply -> TransferSagaCoordinator.on(TransferCreated), events.add(TransferCreated)

TransferCreated -> saga = TransferSaga(BeginTransferSaga command), SagaStore.save(saga), taskScheduler.schedule(timeout)

saga = TransferSaga(command) -> EventSourcedAggregate.apply -> TransferSaga.on(TransferSagaBegan) 여기서 끝김. 이벤트 발행 안됨.

SagaStore.save -> sagaRepository.save, sagaEventRepository.save


```