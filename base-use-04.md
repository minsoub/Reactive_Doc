# Spring Reactive Data로 Reactive Repository 구성하기 

## Getting Started.
- MongoDB Dependency   
MongoDB 코드를 작성하기 위해서, 프로젝트 생성 시에 아래 dependency를 build.gradle 파일에 추가했다.   
```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb-reactive'
```
spring-boot-starter-data-mongodb-reactive는 문서 지향 데이터베이스인 몽고디비와 Spring Data MongoDB Reactive 사요을 위한 스타터이다. 이  dependency는 컴파일 시점에 두 가지 dependency를 가져온다.
- Spring Data MongoDB
- MongoDB's core components + Reactive Stream drivers   

Spring Data는 특정 데이터베이스와 함께 동작하도록 구현된, 많은 하위 프로젝트를 포함하는 프로젝트로, MongoDB 특정 하위 프로젝트를 사용하는 dependency를 가져온다.
<br>
참고. spring-boot-starter-webflux와 spring-boot-starter-data-mongodb-reactive가 모두 Project Reactor를 일시적으로 가져온다는 사실을 아는 것이 중요하다. Spring Boot의 의존성 관리 플러그인이 둘 다 동일한 버전을 사용하도록 보장한다.   

- 쿼리 없이 데이터 작업
일반적인 RDB를 사용할 때에 데이터를 조회하기 위해서는 아래와 같은 쿼리문 작성이 필요하다.
```text
SELECT e
  FROM Employee e
 WHERE e.firstName = :name
```
이 쿼리 코드는 오픈 소스인 Hibernate project에 기반한 JPA(Java Persistence API)로 작성된다.   
그러나 MongoDB를 사용함으로써 아래와 같이 단순하게 데이터를 조회할 수 있다.
```java
interface EmployeeRepository extends ReactiveCrudRepository<Employee, Long>{
   Flux<Employee> findByFirstName(Mono<String> name);
}
```
이 선언적인 인터페이스는 그 어떤 쿼리문의 작성 없이 쿼리문과 동일하게 동작한다.   
또한 Spring Data의 ReactiveCrudRepository를 상속받기 때문에, 그 내부에 선언된 CRUD 동작(save, findById, findAll, delete, deleteById, count, exists, 기타 등등)의 메서드를 사용할 권한을 가진다. 이 부모 클래스에 선언된 메서드 외 적으로도 findByFirstName과 같이 간단한 메서드 시그니처를 작성하는 것만으로 custom finder를 추가할 수 있다.   
Spring Data는 Repository라는 마커 인터페이스를 상속한 인터페이스에 대해서 콘크리트 구현을 생성하고, 이를 이 인터페이스의 메서드 시그니처와 파싱한다. findBy를 만나게 되면, 메서드 이름의 나머지 부분이 FirstName을 도메인 타입으로 인식하여, 속성 이름으로써 추출하기 시작한다.    
이는 Employee가 속성으로써 firstName을 가지고 있기 때문에 가능하며, 쿼리를 만들어 내기에 충분한 정보이다. 마지막으로 Spring Data는 리턴 타입을 보고 어떤 결과를 어셈블 할지 결정한다.    
전체 쿼리는 (쿼리 결과가 아닌) 한번 어셈블되면 캐싱되기 때문에, 쿼리를 여러 번 사용하여도 오버 헤드가 없다.   
- Spring Data Annotation
```java
@Data
@Document
@NoArgsConstructor
@AllArgsContructor
@ToString
public class Image {
   @Id private String id;
   private String name;
   public Image(String id) {
      this.id = id;
   }
}
```
 - @Document: 선택적인 어노테이션으로, 이 도메인이 객체가 MongoDB 컬렉션으로 지정될 것임을 명시. 인자를 넣어 컬렉션명을 지정할 수도 있음. 지정하지 않은 경우 클래스명의 소문자로 지정.
 - @Id: Spring Data Commons의 어노테이션으로 이 속성이 key임을 명시
 
 ## Spring Boot로 Spring Data Repository 연결하기 
 일반적으로 저장소를 연결하려면 도메인 객체와 repository를 정의해야 할 뿐만 아니라, Spring Data를 활성화해야 한다. 각 데이터 저장소는 repository를 지원하기 위해서 활성화해야 하는 어노테이션을 가지고 있다.    
 작성하는 코드에서는 MongoDB의 reactive driver들을 사용하기 때문에 @EnableReactiveRepositories가 필요할 것이다. 하지만 Spring Boot를 사용하면 이러한 작업이 필요 없다. 그 이유는 MongoDB reactive repository 지원을 가능하게 해주는 다음과 같은 코드를 Spring Boot가 스스로 수행하기 때문이다.   
 ```java
 package com.bithumbsystems.webflux.config;
 
 @Configuration
 @ConditionalOnClass({ MongoClient.class,ReactiveMongoRepository.class })
 @ConditionalOnMissingBean({ReactiveMongoRepositoryFactoryBean.class, ReactiveMongoRepositoryConfigurationExtension.class })
 @ConditionalOnProperty(prefix = "spring.data.mongodb.reactive-repositories", name="enabled",havingValue="true", matchIfMissing=true)
 @Import(MongoReactiveRepositoriesAutoConfigureRegisterar.class)
 @AutoConfigureAfter(MongoReactiveDataAutoConfiguration.class)
 public class MongoReactiveRepositoriesAutoConfiguration {
     
 }
```
- @Configuration: 이 클래스가 빈 정의의 source임을 지시
- @ConditionalOnClass: classpath에 있어야 하는 모든 클래스를 나열, 위 코드의 경우에는 MongoDB의 reactiveMongoClient(Reactive Streams version)과 ReactiveMongoRepository. 이는 Reactive MongoDB와 Spring Data MongoDB 2.0이 classpath에 있는 경우에만 적용된다는 것을 의미
- @ConditionalOnMissingBean: ReactiveMongoRepositoryFactoryBean과 ReactiveMongoConfigurationExtension 빈이 없는 경우에만 적용된다는 것을 의미
- @ConditionalOnProperty: 이 속성을 적용하려면 spring.data.mongodb.reactive-repositories 속성을 true로 설정해야 함. (이런 속성이 제공되지 않은 경우 디폴트 세팅)
- @Import: reactive repositoires에 대한 모든 빈 생성을 MongoReactiveRepositoriesAutoConfigureRegistrar에 위임
- @AutoConfigureAfter: autoconfiguration 정책이 MongoReactiveDataAutoConfiguration이 적용된 이후에만 적용됨. 이 경우 특정 인프라가 구성될 수 있음   

spring-boot-starter-mongodb-reactive를 classpath에 추가하였을 때, 이 정책이 시작되었고 MongoDB 데이터베이스와 Reactive하게 상호적용할 수 있는 핵심 빈이 생성되었다.   

## Reactive Repository 생성 
이제 ImageRepository를 생성해보도록 하자.   
```java
package com.bithumbsystems.webflux.repository;

import com.bithumbsystems.webflux.domains.Image;

public interface ImageRepository extends ReactiveCrudRepository<Image, String>{
    Mongo<Image> findByName(String name);
}
```
- ReactiveCrudRepository를 상속받으므로, 위에서 언급한 바와 같이 save, findById, exists, findAll, count, delete, deleteAll 등과 같은 reactive 동작들을 모든 지원되는 Reactor 타입들에 대해 사용 가능
- findByName라는 custom finder를 포함하는데, 이 메서드는 메서드명의 name을 Image의 name 속성에 기반하여 파싱
<br>
ReactiveCrudRepository에서 상속받은 각 동작들은 직접적인 인자나 Reactor-friendly 변형체를 받아들인다. 즉, 우리는 save(Image) 또는 saveAll(Publisher<Image>) 모두 호출 가능하다는 것이다. Mongo나 Flux 둘 다 Publisher를 구현하기 때문에 saveAll()을 사용하여 저장한다.   
우선, Reactive Repository를 사용하기에 앞서, MongoDB 데이터 저장소를 미리 로드해야 한다. 이러한 작업은 실제로 블럭킹 API를 사용하는 것이 좋다. 응용프로그램을 시작할 때, 직접 작성한 로더와 웹 컨테이너 모두 시작되어 스레드가 락 될 위험이 있기 때문이다. Spring Boot는  MongoOperations 객체를 생성하기 때문에, 다음과 같이 간단하게 이 문제를 잡을 수 있다.   
```java
package com.bithumbsystems.webflux.config;

@Component
public class InitDatabase {
    @Bean
    CommandLineRunner init(MongoOperations operations) {
        return args -> {
          operations.dropCollection(Image.class);

          operations.insert(new Image(UUID.randomUUID().toString(), "test1.jpg"));
          operations.insert(new Image(UUID.randomUUID().toString(), "test2.jpg"));
          operations.insert(new Image(UUID.randomUUID().toString(), "test3.jpg"));
          operations.insert(new Image(UUID.randomUUID().toString(), "test4.jpg"));

          operations.findAll(Image.class).forEach(image -> {
             System.out.println(image.toString());
          });
        };
    }
}
```
- @Component: 이 클래스가 Spring Boo에 의해 자동으로 선택되고 bean정의가 있는지 검사
- @Bean: int 메서드가 MongoOperations가 필요한 빈 정의임을 표시. Spring Boot CommandLineRunner를 반환하는데, 이는 applicationContext가 완전히 생성된 후에 실행되는 것을 의미.
- 메서드가 호출되었을 때 command-line runner는 MongoOperations를 사용할 것이고, image 컬렉션의 모든 항목을 삭제하도록 요청. 그런 이후 네 개의 새로운 이미지 레코드를 삽입. 마지막으로 findAll을 사용하여 모든 항목을 반복적으로 콘솔에 출력   

## Mono/Flux 및 연계된 작업을 통해 데이터 가져오기 
Spring Data를 통해 MongoDB와 접속할 repository를 연결하였으니, 이제 우리는 이 repository를 ImageService와 연결해본다.   
가장 먼저, ImageService에 ImageRepository를 주입해주어야 한다.   
```java
@Service
public class ImageService {
    public static String UPLOAD_PATH = "tmp";

    private final ResourceLoader resourceLoader;
    private final ImageRepository imageRepository;

    public ImageService(ResourceLoader resourceLoader, ImageRepository imageRepository) {
        this.resourceLoader = resourceLoader;
        this.imageRepository = imageRepository;
    }
    ...
}
```
ImageService에 ImageRepository 속성을 추가해주고, 생성자에 역시 추가한다.   
<br>
그럼 다음은 ImageService의 메서드들을 수정할 차례이다.   
먼저, findAllImage() 메서드의 경우, 이전에는 이미 업로드된 기존의 파일 이름을 조회하여 Flux<Image> 객체를 만들었다. 하지만 이제 실제 데이터 저장소가 생겼으므로 간단히 모든 데이터를 가져와서 클라이언트에 반환할 수 있다.   
```java
    public Flux<Image> allImages() {
        return imageRepository.findAll();
    }
```
ImageRepository가 findAll() 메서드로 모든 동작을 수행하도록 연결만 해준 것이다.   
<br>
여기서 기억해야 할 점은 findAll은 ReactiveCrudRepository 내에 정의되어 있다는 사실이다. 우리는 이것을 ImageRepository에 작성할 필요가 없다.   
또한 이미지의 Flux가 lazy 하다는 것을 기억하는 것이 좋다. 클라이언트가 요청한 이미지의 수만큼만 어떤 주어진 시간에 나머지 시스템을 통해 데이터베이스로부터 메모리로 가져온다.   
본론적으로 클라이언트 하나 혹은 가능한 많은 것을 요청할 수 있고, reactive 드라이버 덕분에 데이터베이스가 이를 받아들일 수 있다.   
<br>
그럼 다음은 이미지를 업로드하는 메서드를 수정해보자.    
```java
    public Mono<Void> uploadImage(Flux<FilePart> files) {
        return files
                .flatMap(file -> {
                    Mono<Image> saveDatabaseImage = imageRepository.save(
                      new Image(UUID.randomUUID().toString(), file.filename())      
                    );
                    
                    Mono<Void> copyFile = Mono.just(
                                                Paths.get(UPLOAD_PATH, file.filename()).toFile())
                            .log("createImage-picktarget")
                            .map(destFile -> {
                                try {
                                    destFile.createNewFile();
                                    return destFile;
                                }catch(IOException e) {
                                    throw new RuntimeException(e);
                                }
                            })
                            .log("createImage-newfile")
                            .flatMap(file::transferTo)
                            .log("createImage-copy");
                    
                    return Mono.when(saveDatabaseImage, copyFile);
                }).then();
```
- multipart 파일들의 Flux를 가지고, 각각 하나의 파일을 flatMap 하여 두 독립적인 동작으노 나눔: image를 저장하고, 서버에 파일을 복사
- imageRepository를 사용하여 이미지를 MongoDB에 저장하는 Mono와 UUID를 사용하여 고유한 key와 파일명을 만듦  
- WebFlux의 reactive multipart API인 FilePart를 사용하여, 파일을 서버에 복사하는 또 다른 Mono를 빌드
- 이 동작들은 모두 완료되는 것을 보장하기 위해, Mono.when()을 사용하여 두 가지를 함께 이어줌, 즉, 각각의 파일은 MongoDB에 레코드가 쓰이고, 파일이 서버에 복사될 때까지 완료되지 않음.
- 전체 플로우는 then()으로 종료되므로, 모든 파일들이 처리되었을 때 시그널을 보낼 수 있음.   

참고. javascript의 promise를 사용해 보면 Project Reactor의 Mono.when()은 고사양의 promise.all() API와 유사한데, 이 API는 하위 promise들이 모두 완료될 때까지 기다렸다가 진행된다. Project Reactor는 더 많은 동작들이 가능한 아주 뛰어난 promise라고 볼 수 있다. 위의 경우, then()을 ㅏ용하여 여러 연산들을 하나로 묶음으로써, 연산들이 진행되는 플로우를 보장하면서 콜백 지옥을 피할 수 있다.   
참고. FilePart? 보통 흔하게 사용하는 MultipartFile은 본질적으로 블럭킹 서블릿 패러다임과 연결되어 있기 때문에 WebFlux에는 파일 업로드를 Reactive하게 처리한느 새로운 인터페이스인 FilePart가 있다. transferTo() API는 언제 전송할 지 알려주는 Mono<Void>를 반환한다.   
<br>
다음으로 삭제 메서드를 수정한다.   
```java
    public Mono<Void> deleteImage(String filename) {
        Mono<Void> deleteDatabaseImage = imageRepository.findByName(filename).flatMap(imageRepository::delete);
        Mono<Void> deleteFile = Mono.fromRunnable(() -> {
            try {
                Files.deleteIfExists(Paths.get(UPLOAD_PATH, filename));
            }catch(IOException e) {
                throw new RuntimeException(e);
            }
        });
        return Mono.when(deleteDatabaseImage, deleteFile).then();
    }
```
- 먼저 MongoDB의 이미지 레코드를 삭제하기 위한 Mono를 생성. imageRepository를 사용하여 findByName을 수행한 후, imageRepository의 delete를 호출(메서드 레퍼런스 이용)
- 다음으로, Mono.fromRunnable을 사용하여 Mono를 생성. Files.deleteIfExists를 사용하여 파일 삭제. Mono가 호출될 때까지 삭제를 수행되지 않고 지연.
- 이 두 작업을 함께 완료하기 위해 Mono.when()으로 연결
- 결과에 대해서는 관심이 없으므로 then()을 추가하고, then()은 결합된 Mono가 완료되었을 때 완료됨   

수정된 코드를 보면, 연산들을 여러 Mono 정의로 모으는 createImage()와 같은 코딩 패턴이 반복적으로 사용되고, 이 것이 Mono.when()으로 래핑되는 것을 볼 수 있는데, 이를 promise 패턴이라고 한다. reactive 하게 코딩할 때 자주 사용되는 패턴이다.   
<br>
전통적으로 Runnable 객체는 멀티 스레드 방식으로 동작하여 백그라운드에서 실행된다. 이 상황에서 Reactor는 스케줄러를 사용함으로써 어떻게 시작되는지 완전히 제어한다. Reactor는 Runnable 객체가 작업을 완료했을 때 reactive stream 완료 시그널이 발생하도록 할 수 있다. 이는 Project Reactor의 다양한 작업들의 요점이다. 원하는 상태로 선언하고, 모든 작업 스케줄링과 스레드 관리를 프레임워크가 수행하도록 한다.   

