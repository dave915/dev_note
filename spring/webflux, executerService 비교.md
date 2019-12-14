### webflux, executorService 비교

# 이슈

원인

호텔 리스트의 실시간 방정보 조회시 벤더사 수 만큼 parallelStream으로 돌면서 api 요청.

api 요청시 restTemplate의 read timeout 은 5초로 잡고 있었지만,  parallelStream이 동작 할 수 있는 thread가 부족 해서 api요청을 못하고 홀딩 되어 있었음.

parallelStream이 사용하는 thread pool은 common thread pool 이며, 기본적으로 cpu 코어 수 만큼 생성됨.

- 코드는 임의로 수정했기 때문에 돌려봐도 정상동작 안함

수정 전
```java
// git에 올리기 위해 코드 임의로 수정
public List<HotelInformation> getHotels(Request request) {

        // 호텔리스트 조회

        // 이슈가 되는 부분 (벤더사 방 정보 조회)
        vendors.parallelStream().forEach(vendor -> proxy.blockAvailability());


        // 캐시 저장
    }
```

# 해결 방법 논의

1. webflux 적용
2. 해당 api 요청 용 thread pool을 이용

## Test 진행

**webflux 코드**

```java
// git에 올리기 위해 코드 임의로 수정
public Mono<List<HotelInformation>> getHotelsTestWebflux(Request request) {
        Mono<List<Vendor>> vendorMono = Mono.fromCallable(this::getVendorList);
        Mono<List<Promotion>> PromotionMono = Mono.fromCallable(() -> getPromotions());

        return Mono.zip(vendorInfoMono, savingsPromotionMono)
                .flatMap(tuple -> {
                    List<Vendor> vendors = tuple.getT1();
                    List<Promotion> promotions = tuple.getT2();

                    return proxy.getHotelsWebclient(request)
                            .flatMap(HotelProxyDtoList -> mappingAvailableRooms(request, vendor, HotelProxyDtoList))
                            .map(HotelProxyDto -> makeHotelInformationList(request, vendor, promotions, HotelProxyDto))
                            .log();
                });
    }
```

**executorService 코드**

```java
// git에 올리기 위해 코드 임의로 수정
public List<HotelInformation> getHotelsTestExecutorService(Request request) {
        List<Vendor> vendors = getVendorList();
        List<HotelProxyDto> hotelProxyDtos = proxy.getHotels(request);
        setRooms(request, hotelProxyDto, vendors);

        List<HotelInformation> hotelInformations = hotelProxyDtos.stream()
                .map(HotelProxyDto::testConvertHotelInformation)
                .collect(Collectors.toList());

        return hotelInformations;
    }

    private void setRooms(Request request, List<ProxyHotelDto> hotelProxyDtos, List<Vendor> vendors) {
        try {
            CompletableFuture.allOf(vendors.stream()
                    .map(vendor -> CompletableFuture.supplyAsync(() -> testAllstayHotelProxy.blockAvailability(request, vendor, hotelProxyDtos), roomExecutorService))
                    .toArray(CompletableFuture[]::new)).get();
        } catch (InterruptedException | ExecutionException e) {
            log.error("방 정보 셋팅 중 에러 : {}", e.getMessage());
        }
    }
```

## 테스트 방법

벤더사 방정보 요청 하는 api는 직접 요청 하지 않고, Webflux 테스트 시에는 Mono delay 를 이용해 5초 대기

ExecutorService 테스트 시에는 Thread.sleep 를 이용해 5초 대기

nGrinder 설정

![](/images/spring/nGrinder_setting.png)

## 테스트 결과

### nGrinder 결과

![](/images/spring/nGrinder_webflux.png)

![](/images/spring/nGrinder_executor.png)

### 리소스 사용

VisualVm을 이용해서 확인.

VisualVm download : [https://visualvm.github.io/download.html](https://visualvm.github.io/download.html)

intellij 연동 : [https://ryudung.tistory.com/26](https://ryudung.tistory.com/26)

**Webflux**

![](/images/spring/visualvm_webflux.png)

**ExecutorService**

![](/images/spring/visualvm_executor.png)

# 결론

전체적인 테스트 실행수, TPS등 성능은 Webflux가 더 좋았지만,

코드 수정량 대비 성능 향상, 내부결제 시 반영될 코드 까지 고려해서 우선은 ExecutorService를 사용하는 방향으로 결정.
