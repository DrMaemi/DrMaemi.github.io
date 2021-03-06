---
title: '[회고] Spring Boot, JPA를 이용해 셀러 광고 플랫폼 API 서버 만들기'
uml: true
author_profile: true
toc_label: '[회고] Spring Boot, JPA를 이용해 셀러 광고 플랫폼 API 서버 만들기'
last_modified_at: 2022-07-09 23:06:14 +0900
---

<figure data-ke-type="opengraph"><a href="https://github.com/drmaemi/Ad-platform" data-source-url="https://github.com/drmaemi/Ad-platform">
<div class="og-image" style="background-image: url('https://drive.google.com/uc?export=view&id=1TEGMLaj66tqxG4V9a1YGwPrrQL6E6Fn6');"></div>
<div class="og-text">
<p class="og-title">셀러 광고플랫폼 구축</p>
<p class="og-desc">커머스 도메인에서 셀러의 업체 입점부터 광고입찰 생성, FRONT 전시, CPC 방식의 클릭과금, 과금데이터의 배치성 정산집계까지 일련의 프로세스를 수행할 수 있는 서버 개발</p>
<p class="og-host">https://github.com/drmaemi</p></div></a></figure>

## 들어가며, 프로젝트 시작 전에
최근 모 기업의 과제로 셀러 광고 플랫폼 API 서버를 구축하는 프로젝트를 진행했습니다. 과제는 1. CRUD 기능을 가진 상품정보 API 서버 구축(기술 스택 무관), 2. 셀러 광고 플랫폼 구축(스프링부트와 JPA, 인메모리 DB, Spring REST Docs, Spring Batch 기술스택 활용), 3. 장바구니, 상품 구매 경험을 할 수 있는 웹 프로토타입 구축(기술 스택 무관) 세 개 중 하나를 선택해서 진행했어야 하는데, 결과적으로 2번 과제를 선택해서 진행했습니다. 1번 과제는 흔히 해볼 수 있는 다소 식상한 주제라서 제외했고, 2번과 3번 과제 중 고민했는데 과제 요구 사항을 보고 백엔드 기술에 좀 더 집중할 수 있는 2번 과제를 선택했습니다.

2번 과제에 필요한 기술 스택에 대해서, 스프링부트와 JPA를 이용한 소규모 웹 어플리케이션 토이 프로젝트를 해본 경험이 있는데 그 외에 나머지 기술 스택에 대해서는 다뤄본 적이 없고 검색해보니 한 번 경험해보고 싶은 매우 흥미로운 것들이라 생각이 들어서 과감하게 도전했습니다.

결과적으로 경험해보지 못한 기술 스택을 사용하며 정말 많은 것들을 배울 수 있었습니다. 특히 Spring REST Docs를 이용한 TDD, ORM 프레임워크 JPA와 관계형 DB 설계, N+1 성능 개선, Spring Batch 처리, 복잡한 쿼리를 위한 Native Query와 Type-safe한 QueryDSL, 비즈니스와 관련된 복잡한 유효성 검증에 대해서 고민해볼 수 있는 기회였고 포스트에서 다룰 회고 또한 이런 부분들을 중심적으로 서술해나갈 생각입니다.

## 도메인 분석 단계
개발환경과 화면/기능 요구사항은 다음과 같았습니다.

<p class=short><b>개발 환경</b></p>

- JAVA 8 이상
- Spring Boot 사용
- DB 컨트롤은 JPA or Mybatis mapper 방식으로 구현
- Gradle 또는 Maven 기반
- 인메모리 DB(ex. H2) 사용
- Swagger 또는 Spring Rest Docs를 사용한 API Documentation
- 외부 라이브러리 및 오픈소스 사용 가능 *README 파일에 사용한 오픈 소스와 사용 목적 명시

<p class=short><b>개발 목표 및 화면/기능 요구사항</b></p>

- 개발 목표
    - e커머스 도메인의 업체 입점부터 광고 입찰생성, FRONT 전시, CPC 방식의 클릭과금, 과금데이터 배치성 정산집계까지 일련의 프로세스를 구축

- 업체/계약 도메인
    - 업체생성
        - 상품에 셋팅된 업체만 등록 가능
        - ...
        - 사업자번호, 업체전화번호 등의 Data 셋팅 시 자릿수, 숫자여부만 Validation
    - 계약생성
        - 업체생성을 통해 업체 Master 정보가 생성된 업체에 안해 업체계약 가능
        - ...
        - 업체별 계약 기간 중복이 없도록 유효성 검사
- 광고 도메인
    - 광고입찰
        - 계약생성, 계약기간이 유효한 업체에 한해 입찰 가능
        - ...
    - 광고전시
        - 입찰된 광고 중 입찰가 기준 상위 3개 입찰을 노출 대상으로 Response
        - 상품정보 기준 재고수량 0인 경우 노출 대상 제외
        - ...
    - 광고과금
        - CPC 방식 과금을 가정하여 광고전시 API를 통해 노출된 광고가 클릭된 경우 과금 생성
- 정산 도메인
    - 광고과금대상 데이터 기준 일별 정산 집계 데이터 생성
    - ...

크게 세 개의 도메인으로 구성되어 있는데, 요구사항을 보니 데이터가 서로 연관되어 있으며 연관된 데이터의 유효성 검증 요소가 많이 보이기 때문에 관계형 데이터 모델을 사용하면 스프링부트의 유효성 검증 기능과 함께 DB레벨에서, 어플리케이션 레벨에서 데이터 무결성을 유지할 수 있을 것이라 생각이 들었습니다. 그에 따라 프로세스 흐름에 따른 데이터의 정합성도 보장되리라 생각했습니다.

일단 본격적인 개발을 진행하기 앞서 DB 설계를 진행하고자 ERD를 작성했습니다. 작성된 ERD는 아래 <그림 1>과 같습니다.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent; margin:auto;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom lightbox&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2022-06-19T10:30:59.686Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36\&quot; etag=\&quot;Fu0R_fx8nbonyQTN7Q4v\&quot; version=\&quot;20.0.1\&quot; type=\&quot;google\&quot;&gt;&lt;diagram id=\&quot;9su41jI4dY5pf6SDruzu\&quot; name=\&quot;ERD\&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
&lt;그림 1. 광고 플랫폼 도메인 ERD&gt;
{: style="text-align: center;"}

각 기능에 대응되는 데이터를 테이블로 설계해서, 결과적으로 업체(COMPANY), 상품(PRODUCT), 계약(CONTRACT), 광고입찰(ADVERTISEMENT_BID), 광고전시(ADVERTISEMENT_DISPLAY), 광고과금(ADVERTISEMENT_CHARGE), 광고과금정산(ADVERTISEMENT_CHARGE_CAL) 총 6개의 테이블과 1개의 뷰를 설계했고, 외래키를 이용해 참조무결성을 보장하여 상품 리스트에 셋팅된 업체에 한해서만 업체 등록을 가능케하는 등 유효성 검증을 DB 레벨에서 할 수 있도록 했습니다. 과금 데이터에 대해 일별 배치 집계 후 정산된 데이터를 저장할 광고과금정산 테이블은 클릭일자-광고입찰ID 두 개의 필드를 이용해 복합키를 사용했습니다.

여기까지는 관계형 DB를 공부하신 분들이라면 조금 고민한 후 무리없이 잘 진행할 수 있는데, 문제는 이를 어떻게 ORM 프레임워크 JPA를 이용해 구현할 것인가에 대해서는 JPA의 연관 관계 설계에 대한 이해가 필요했습니다. 저 또한 이 부분에 대해 자세히 알고 있지 않은 상태에서 프로젝트를 진행하다 보니 별도의 스터디가 필요했고, 구현 가능 여부가 확실치 않아서 프로젝트 진행 최우선 목표로 JPA 코드를 통한 데이터 제어 기능 구현을 삼았습니다.

## JPA를 이용한 테이블 연관 관계 설계
ERD 상 외래 키로 참조 관계를 가지고 있는 모든 테이블은 JPA를 통해 연관 관계를 설계해줘야 할 대상입니다. 모든 연관 관계를 다루기엔 포스트에서 다룰 내용이 지나치게 많아질테니, 일대다 연관 관계에 대해서만 생각해보면 업체-상품, 광고입찰-과금이 있습니다. 하나의 업체가 다양한 상품을 등록할 수 있고, 광고입찰로 노출된 광고를 다수의 사람이 클릭하여 다수의 과금 데이터를 생성시킬 수 있습니다.

그런데 JPA에는 일대다 연관 관계에 대해 단방향, 양방향 개념이 있습니다. 관계형 데이터 모델을 객체 지향 패러다임으로 다룰 수 있도록 하는 ORM 프레임워크인 JPA는, 데이터를 객체 관점에서 바라보기 때문에 객체 간 참조 관계에 따라 단방향과 양방향이 있을 수 있습니다. 이 때 참조 관계는 비즈니스 로직에 의존한다고 생각이 들었습니다. 데이터를 조작할 때 연관 관계가 있는 두 객체가 서로 참조할 필요가 있는가, 혹은 한 객체에서 다른 객체를 참조하고 그 반대는 필요하지 않은가 이런 필요성에 의해 양방향이냐 단방향이냐를 결정하여 설계했고, 그 결과 업체-상품의 관계는 양방향으로, 그 외에 경우는 단방향으로 설계했습니다.

<p class=short>CompanyEntity, ProductEntity 코드</p>

```java
/**
 * 엔티티의 생명주기를 감시하는 JpaAuditing 기능을 수행하는 엔티티 클래스
 * DB에 매핑되는 테이블 없이, 해당 엔티티를 상속하는 자식 엔티티에 대해 JpaAuditing 기능 수행
 */
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class TimeEntity {
    @CreatedDate
    @Column(updatable=false, nullable=false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    @Column(nullable=false)
    private LocalDateTime lastModifiedDate;
}

/**
 * DB 테이블 'COMPANY'에 매핑되는 엔티티 클래스
 * ProductEntity와 일대다 양방향 연관 관계
 */
@Getter
@Setter
@NoArgsConstructor(access= AccessLevel.PROTECTED)
@Entity
@Table(name="COMPANY")
public class CompanyEntity extends TimeEntity implements Serializable {
    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    @Column(length=30, unique=true, nullable=false, updatable=false)
    private String name;

    @Column(length=10)
    private String businessRegistrationNumber;

    @Column(length=20)
    private String phoneNumber;

    @Column(length=50)
    private String address;

    @OneToMany(mappedBy="companyEntity", fetch=FetchType.LAZY)
    private List<ProductEntity> productEntities;
    ...

/**
 * DB 테이블 'PRODUCT'에 매핑되는 엔티티 클래스
 * CompanyEntity와 다대일 양방향 연관 관계
 */
@Getter
@Setter
@NoArgsConstructor(access= AccessLevel.PROTECTED)
@Entity
@Table(name="PRODUCT")
public class ProductEntity extends TimeEntity {
    @Id
    @GeneratedValue(strategy= GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch=FetchType.LAZY)
    @JoinColumn(name="company_name", referencedColumnName="name")
    private CompanyEntity companyEntity;

    @Column(nullable=false)
    private String productName;

    @Column(nullable=false)
    private Long price;

    @Column(nullable=false)
    private Long stock;
    ...
```

또한 JPA의 영속성 문맥에 있는 엔티티 객체의 노출을 최소화하여 데이터 오류를 방지하기 위해 DTO(Data Transfer Object) 클래스로 엔티티를 감싸도록 했습니다.

<p class=short>샘플 데이터</p>

상품 ID | 업체명 | 상품명 | 판매가 | 재고
:----: | :----: | :----: | :----: | :----:
1000000001 | 이마트 | 허쉬 초코멜로쿠키 45g | 600 | 10
1000000002 | 신세계백화점 | 크리스피롤 12 곡 180g | 2,800 | 5
1000000003 | 삼성전자 | 갤럭시 22 자급제모델 | 1,200,000 | 0
1000000004 | 이마트 | 말랑카우 핸드워시 250ml | 2,600 | 6
1000000005 | 신세계백화점 | 삼립 미니꿀호떡 322g | 1,200 | 4
1000000006 | 이마트 | 피코크 어랑손만두 어랑만두 700g | 6,400 | 2
1000000007 | 신세계백화점 | 뺴빼로바 아몬드아이스크림 4입 | 2,800 | 1
1000000008 | 이마트 | 피코크 에이 클래스 우유 900ml | 2,500 | 3
1000000009 | 이마트 | 아삭달콤 방울토마토 500g | 4,500 | 0
1000000010 | 이마트 | [롯데삼강] 돼지바 (70ml*6입) | 3,000 | 1
1000000011 | 나이키 | 나이키 덩크로우 흰검 | 129,000 | 4
1000000012 | 스타벅스 | 이달의 원두 500g | 18,500 | 4
1000000013 | 스타벅스 | 아쿠아 머그 | 23,000 | 7
1000000014 | 삼성전자 | 삼성전자 43인치 스마트모니터 | 480,000 | 2
1000000015 | 나이키 | 나이키 헤리티지 스우시 캡 | 25,000 | 5
{: .align-center}

![](https://drive.google.com/uc?export=view&id=1OumEfcTfKXUo_K443Q7gRlEWZh8GpyLL){: .align-center}
&lt;화면 1. 상품-업체 샘플 데이터 생성 결과&gt;
{: style="text-align: center;"}


그런데 양방향 일대다 연관 관계인 업체-상품 엔티티를 코딩해놓고, 샘플 데이터를 생성한 뒤 조회해보니 많은 문제가 발생했습니다.

### 양방향 연관 관계 순환 참조
테스트를 위해 생성된 데이터에서 업체 정보 리스트를 조회하려고 시도했더니 다음과 같은 오류가 발생했습니다.

```txt
java.lang.StackOverflowError: null
    at java.base/java.util.stream.AbstractPipeline.<init>(AbstractPipeline.java:211) ~[na:na]
    at java.base/java.util.stream.ReferencePipeline.<init>(ReferencePipeline.java:96) ~[na:na]
    at java.base/java.util.stream.ReferencePipeline$StatelessOp.<init>(ReferencePipeline.java:800) ~[na:na]
    at java.base/java.util.stream.ReferencePipeline$3.<init>(ReferencePipeline.java:191) ~[na:na]
    at java.base/java.util.stream.ReferencePipeline.map(ReferencePipeline.java:190) ~[na:na]
    at com.ssg.backendpreassignment.entity.CompanyEntity.toDto(CompanyEntity.java:58) ~[classes/:na]
    at com.ssg.backendpreassignment.entity.ProductEntity.toDto(ProductEntity.java:47) ~[classes/:na]
    at com.ssg.backendpreassignment.entity.CompanyEntity.lambda$toDto$0(CompanyEntity.java:58) ~[classes/:na]
    at java.base/java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:197) ~[na:na]
    at java.base/java.util.Iterator.forEachRemaining(Iterator.java:133) ~[na:na]
    at java.base/java.util.Spliterators$IteratorSpliterator.forEachRemaining(Spliterators.java:1845) ~[na:na]
    at java.base/java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:509) ~[na:na]
    at java.base/java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:499) ~[na:na]
    at java.base/java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:921) ~[na:na]
    at java.base/java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234) ~[na:na]
    at java.base/java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:682) ~[na:na]
    at com.ssg.backendpreassignment.entity.CompanyEntity.toDto(CompanyEntity.java:58) ~[classes/:na]
    at com.ssg.backendpreassignment.entity.ProductEntity.toDto(ProductEntity.java:47) ~[classes/:na]
    at com.ssg.backendpreassignment.entity.CompanyEntity.lambda$toDto$0(CompanyEntity.java:58) ~[classes/:na]
    ...
```

스택오버플로우가 발생한 걸 보니 호출 스택이 초과한 것으로 보입니다. JPA가 업체 정보를 조회하면서 상품 정보를 조회하고, 상품 정보를 조회하면서 업체 정보를 조회하면서 다시 상품 정보를 조회하는 식으로 무한히 참조하는 것이 문제였습니다. 양방향 연관 관계 설계 시 발생할 수 있는 문제로 어떻게 해결할까 고민하다가, 엔티티를 DTO로 감쌀 때 자신이 참조하고 있는 연관 엔티티는 DTO로 감싸지 않고 null로 설정하도록 구현하여 해결했습니다.

<p class=short>엔티티 순환 참조 해결 코드</p>

```java
/**
 * DB 테이블 'COMPANY'에 매핑되는 엔티티 클래스
 * ProductEntity와 일대다 양방향 연관 관계
 */
@Getter
@Setter
@NoArgsConstructor(access= AccessLevel.PROTECTED)
@Entity
@Table(name="COMPANY")
public class CompanyEntity extends TimeEntity implements Serializable {
    ...

    public CompanyDto toDto() {
        return CompanyDto.builder()
                .id(this.getId())
                .name(this.getName())
                .businessRegistrationNumber(this.getBusinessRegistrationNumber())
                .phoneNumber(this.getPhoneNumber())
                .address(this.getAddress())
                .productDtos(this.getProductEntities().stream().map(productEntity -> productEntity.toDtoExceptCompany()).collect(Collectors.toList()))
                .createdDate(this.getCreatedDate())
                .lastModifiedDate(this.getLastModifiedDate())
                .build();
    }

    public CompanyDto toDtoExceptProducts() {
        return CompanyDto.builder()
                .id(this.getId())
                .name(this.getName())
                .businessRegistrationNumber(this.getBusinessRegistrationNumber())
                .phoneNumber(this.getPhoneNumber())
                .address(this.getAddress())
                .createdDate(this.getCreatedDate())
                .lastModifiedDate(this.getLastModifiedDate())
                .build();
    }
}

/**
 * DB 테이블 'PRODUCT'에 매핑되는 엔티티 클래스
 * CompanyEntity와 다대일 양방향 연관 관계
 */
@Getter
@Setter
@NoArgsConstructor(access= AccessLevel.PROTECTED)
@Entity
@Table(name="PRODUCT")
public class ProductEntity extends TimeEntity {
    ...

    public ProductDto toDto() {
        return ProductDto.builder()
                .id(this.getId())
                .companyDto(this.getCompanyEntity().toDtoExceptProducts())
                .productName(this.getProductName())
                .price(this.getPrice())
                .stock(this.getStock())
                .createdDate(this.getCreatedDate())
                .lastModifiedDate(this.getLastModifiedDate())
                .build();
    }

    public ProductDto toDtoExceptCompany() {
        return ProductDto.builder()
                .id(this.getId())
                .productName(this.getProductName())
                .price(this.getPrice())
                .stock(this.getStock())
                .createdDate(this.getCreatedDate())
                .lastModifiedDate(this.getLastModifiedDate())
                .build();
    }
}
```

### N+1 문제
순환 참조 문제를 해결하고 기업 정보 리스트 조회가 잘 되길래 좋아하고 있었는데, 콘솔에 출력되는 SQL 쿼리문이 생각보다 많아서 유심히 보다가 문제가 있음을 확인했습니다. 생성된 쿼리문은 아래와 같습니다.

```txt
Hibernate: select companyent0_.id as id1_4_, companyent0_.created_date as created_2_4_, companyent0_.last_modified_date as last_mod3_4_, companyent0_.address as address4_4_, companyent0_.business_registration_number as business5_4_, companyent0_.name as name6_4_, companyent0_.phone_number as phone_nu7_4_ from company companyent0_
Hibernate: select productent0_.company_name as company_7_6_0_, productent0_.id as id1_6_0_, productent0_.id as id1_6_1_, productent0_.created_date as created_2_6_1_, productent0_.last_modified_date as last_mod3_6_1_, productent0_.company_name as company_7_6_1_, productent0_.price as price4_6_1_, productent0_.product_name as product_5_6_1_, productent0_.stock as stock6_6_1_ from product productent0_ where productent0_.company_name=?
Hibernate: select companyent0_.id as id1_4_1_, companyent0_.created_date as created_2_4_1_, companyent0_.last_modified_date as last_mod3_4_1_, companyent0_.address as address4_4_1_, companyent0_.business_registration_number as business5_4_1_, companyent0_.name as name6_4_1_, companyent0_.phone_number as phone_nu7_4_1_, productent1_.company_name as company_7_6_3_, productent1_.id as id1_6_3_, productent1_.id as id1_6_0_, productent1_.created_date as created_2_6_0_, productent1_.last_modified_date as last_mod3_6_0_, productent1_.company_name as company_7_6_0_, productent1_.price as price4_6_0_, productent1_.product_name as product_5_6_0_, productent1_.stock as stock6_6_0_ from company companyent0_ left outer join product productent1_ on companyent0_.name=productent1_.company_name where companyent0_.name=?
Hibernate: select productent0_.company_name as company_7_6_0_, productent0_.id as id1_6_0_, productent0_.id as id1_6_1_, productent0_.created_date as created_2_6_1_, productent0_.last_modified_date as last_mod3_6_1_, productent0_.company_name as company_7_6_1_, productent0_.price as price4_6_1_, productent0_.product_name as product_5_6_1_, productent0_.stock as stock6_6_1_ from product productent0_ where productent0_.company_name=?
Hibernate: select companyent0_.id as id1_4_1_, companyent0_.created_date as created_2_4_1_, companyent0_.last_modified_date as last_mod3_4_1_, companyent0_.address as address4_4_1_, companyent0_.business_registration_number as business5_4_1_, companyent0_.name as name6_4_1_, companyent0_.phone_number as phone_nu7_4_1_, productent1_.company_name as company_7_6_3_, productent1_.id as id1_6_3_, productent1_.id as id1_6_0_, productent1_.created_date as created_2_6_0_, productent1_.last_modified_date as last_mod3_6_0_, productent1_.company_name as company_7_6_0_, productent1_.price as price4_6_0_, productent1_.product_name as product_5_6_0_, productent1_.stock as stock6_6_0_ from company companyent0_ left outer join product productent1_ on companyent0_.name=productent1_.company_name where companyent0_.name=?
Hibernate: select productent0_.company_name as company_7_6_0_, productent0_.id as id1_6_0_, productent0_.id as id1_6_1_, productent0_.created_date as created_2_6_1_, productent0_.last_modified_date as last_mod3_6_1_, productent0_.company_name as company_7_6_1_, productent0_.price as price4_6_1_, productent0_.product_name as product_5_6_1_, productent0_.stock as stock6_6_1_ from product productent0_ where productent0_.company_name=?
Hibernate: select companyent0_.id as id1_4_1_, companyent0_.created_date as created_2_4_1_, companyent0_.last_modified_date as last_mod3_4_1_, companyent0_.address as address4_4_1_, companyent0_.business_registration_number as business5_4_1_, companyent0_.name as name6_4_1_, companyent0_.phone_number as phone_nu7_4_1_, productent1_.company_name as company_7_6_3_, productent1_.id as id1_6_3_, productent1_.id as id1_6_0_, productent1_.created_date as created_2_6_0_, productent1_.last_modified_date as last_mod3_6_0_, productent1_.company_name as company_7_6_0_, productent1_.price as price4_6_0_, productent1_.product_name as product_5_6_0_, productent1_.stock as stock6_6_0_ from company companyent0_ left outer join product productent1_ on companyent0_.name=productent1_.company_name where companyent0_.name=?
Hibernate: select productent0_.company_name as company_7_6_0_, productent0_.id as id1_6_0_, productent0_.id as id1_6_1_, productent0_.created_date as created_2_6_1_, productent0_.last_modified_date as last_mod3_6_1_, productent0_.company_name as company_7_6_1_, productent0_.price as price4_6_1_, productent0_.product_name as product_5_6_1_, productent0_.stock as stock6_6_1_ from product productent0_ where productent0_.company_name=?
Hibernate: select companyent0_.id as id1_4_1_, companyent0_.created_date as created_2_4_1_, companyent0_.last_modified_date as last_mod3_4_1_, companyent0_.address as address4_4_1_, companyent0_.business_registration_number as business5_4_1_, companyent0_.name as name6_4_1_, companyent0_.phone_number as phone_nu7_4_1_, productent1_.company_name as company_7_6_3_, productent1_.id as id1_6_3_, productent1_.id as id1_6_0_, productent1_.created_date as created_2_6_0_, productent1_.last_modified_date as last_mod3_6_0_, productent1_.company_name as company_7_6_0_, productent1_.price as price4_6_0_, productent1_.product_name as product_5_6_0_, productent1_.stock as stock6_6_0_ from company companyent0_ left outer join product productent1_ on companyent0_.name=productent1_.company_name where companyent0_.name=?
Hibernate: select productent0_.company_name as company_7_6_0_, productent0_.id as id1_6_0_, productent0_.id as id1_6_1_, productent0_.created_date as created_2_6_1_, productent0_.last_modified_date as last_mod3_6_1_, productent0_.company_name as company_7_6_1_, productent0_.price as price4_6_1_, productent0_.product_name as product_5_6_1_, productent0_.stock as stock6_6_1_ from product productent0_ where productent0_.company_name=?
Hibernate: select companyent0_.id as id1_4_1_, companyent0_.created_date as created_2_4_1_, companyent0_.last_modified_date as last_mod3_4_1_, companyent0_.address as address4_4_1_, companyent0_.business_registration_number as business5_4_1_, companyent0_.name as name6_4_1_, companyent0_.phone_number as phone_nu7_4_1_, productent1_.company_name as company_7_6_3_, productent1_.id as id1_6_3_, productent1_.id as id1_6_0_, productent1_.created_date as created_2_6_0_, productent1_.last_modified_date as last_mod3_6_0_, productent1_.company_name as company_7_6_0_, productent1_.price as price4_6_0_, productent1_.product_name as product_5_6_0_, productent1_.stock as stock6_6_0_ from company companyent0_ left outer join product productent1_ on companyent0_.name=productent1_.company_name where companyent0_.name=?
```

업체 리스트를 조회하는 요청인데, 업체와 연관된 상품 정보에 대한 SELECT 쿼리가 실행된 것을 확인할 수 있었습니다. 이 문제는 그 유명한 N+1 문제였는데, N+1 문제의 현상과 해결책에 대해서 조사를 하다가 다른 사람들의 포스트에서 문제 삼는 쿼리 현상과 제가 겪은 쿼리 현상이 조금 달라 의아했습니다. 다른 사람들의 쿼리 현상과 제 쿼리 현상을 비교하면 다음과 같습니다.

다른 사람들의 쿼리 현상 | 나의 쿼리 현상
:---: | :---:
일대다의 '일'에 해당하는 엔티티를 조회하는 쿼리 1번, '다'에 해당하는 연관 엔티티(N개)를 조회하는 쿼리 N번 발생(1+N) | 일대다의 '일'에 해당하는 엔티티를 전체 조회하는 쿼리 1번, '다'에 해당하는 연관 엔티티를 조회할 때 쿼리 1번, FK에 대해 LEFT OUTER JOIN 쿼리 1번 발생(1+1+1)
쿼리 N번은 각각 JOIN 없이 WHERE절 조건에 FK를 이용하여 SELECT(같은 쿼리가 중복하여 발생하는 것으로 보임) | '다' 쿼리 시 똑같이 JOIN 없이 WHERE절 조건에 FK를 이용하여 SELECT 하되 중복 쿼리가 없음
LEFT OUTER JOIN문이 보이지 않음 | LEFT OUTER JOIN문이 보임

LEFT OUTER JOIN은 테스트 코드로 실험 결과 단방향 일대다 연관 관계에서는 발생하지 않고 양방향 일대다 연관 관계에서만 발생하는 것으로 확인했습니다. 그리고 제가 참고했던 포스트들은 2018년도~2021년도 자료가 많았는데, 제 추측으론 최근에 JPA Hibernate 진영에서 N+1 문제에 대한 이슈가 계속 제기되자 이에 대한 성능 개선을 한 것 같습니다.

쿼리문을 더 자세히 보기 위해 업체 리스트가 아니라 업체 ID, 이름에 따라 업체 한 개에 대해서 정보를 조회하는 테스트 코드를 작성해서 쿼리를 확인했습니다.

<p class=short>컨트롤러</p>

```java
/**
 * 업체 도메인 관련 HTTP request 매핑 및 처리를 위한 컨트롤러 클래스
 */
@RestController
@RequiredArgsConstructor
public class CompanyRestController {
    private final CompanyService companyService;

    @GetMapping("/api/company/id/{companyId}")
    public ResponseEntity<?> getCompanyById(@PathVariable Long companyId) {
        return new ResponseEntity<RestResponse>(RestResponse.builder()
                .code(HttpStatus.OK.value())
                .status(HttpStatus.OK)
                .result(companyService.findById(companyId))
                .build(), HttpStatus.OK);
    }

    @GetMapping("/api/company/name/{companyName}")
    public ResponseEntity<?> getCompanyByName(@PathVariable String companyName) {
        return new ResponseEntity<RestResponse>(RestResponse.builder()
                .code(HttpStatus.OK.value())
                .status(HttpStatus.OK)
                .result(companyService.findByName(companyName))
                .build(), HttpStatus.OK);
    }
}
```

<p class=short>서비스</p>

```java
/**
 * 업체 도메인의 비즈니스 로직을 처리하는 서비스 클래스
 */
@Service
@RequiredArgsConstructor
public class CompanyService {
    private final ProductRepository productRepository;
    private final CompanyRepository companyRepository;

    @Transactional(readOnly=true)
    public CompanyDto findById(Long id) {
        Optional<CompanyEntity> companyEntityWrapper = companyRepository.findById(id);

        if (companyEntityWrapper.isPresent()) {
            return companyEntityWrapper.get().toDto();
        }

        return null;
    }

    @Transactional
    public CompanyDto findByName(String companyName) {
        Optional<CompanyEntity> companyEntityWrapper = companyRepository.findByName(companyName);
        
        if (companyEntityWrapper.isPresent()) {
            return companyEntityWrapper.get().toDto();
        }
        
        return null;
    }
    ...
}
```

<p class=short>JPA 인터페이스(JpaRepository)</p>

```java
/**
 * CompanyEntity의 영속성을 관리하며 엔티티에 매핑된 DB 테이블 'COMPANY'에 CRUD 기능 수행
 */
public interface CompanyRepository extends JpaRepository<CompanyEntity, Long> {
    Optional<CompanyEntity> findByName(String name);
}
```

<p class=short>테스트 코드</p>

```java
@SpringBootTest
@AutoConfigureMockMvc
class ApplicationTests {
    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private CompanyService companyService;

    @Test
    void getCompanyById() throws Exception {
        Long companyId = (long)1e9+1;
        String companyName = "이마트";

        this.mockMvc.perform(
                        get("/api/company/id/"+companyId)
                                .contentType(MediaType.APPLICATION_JSON)
                )
                .andExpect(status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.result.name").value("이마트"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.result.productDtos.size()").value(6));
    }

    @Test
    void getCompanyByName() throws Exception {
        Long companyId = (long)1e9+1;
        String companyName = "이마트";

        this.mockMvc.perform(
                        get("/api/company/name/"+companyName)
                                .contentType(MediaType.APPLICATION_JSON)
                )
                .andExpect(status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.result.id").value(companyId))
                .andExpect(MockMvcResultMatchers.jsonPath("$.result.productDtos.size()").value(6));
    }
}
```

![](https://drive.google.com/uc?export=view&id=10ZsG6CMXPh1315O8DRQLxg4sy_bQ2SHO){: .align-center}
&lt;화면 2. getCompanyById(), getCompanyByName() API 테스트 결과(no fetch)&gt;
{: style="text-align: center;"}

소요 시간을 확인해보니 getCompanyById()가 getCompanyByName()에 비해 약 10배가량 더 소요했는데, 이는 테스트 진행 시 필요한 Mock 빈들을 주입받는 시간으로 보입니다. 이에 대해서도 자세한 테스트는 나중에 다뤄보기로 하고, 우선 쿼리를 살펴보기로 했습니다.

```txt:getCompanyById()
Hibernate: 
    select
        companyent0_.id as id1_4_0_,
        companyent0_.created_date as created_2_4_0_,
        companyent0_.last_modified_date as last_mod3_4_0_,
        companyent0_.address as address4_4_0_,
        companyent0_.business_registration_number as business5_4_0_,
        companyent0_.name as name6_4_0_,
        companyent0_.phone_number as phone_nu7_4_0_,
        productent1_.company_name as company_7_6_1_,
        productent1_.id as id1_6_1_,
        productent1_.id as id1_6_2_,
        productent1_.created_date as created_2_6_2_,
        productent1_.last_modified_date as last_mod3_6_2_,
        productent1_.company_name as company_7_6_2_,
        productent1_.price as price4_6_2_,
        productent1_.product_name as product_5_6_2_,
        productent1_.stock as stock6_6_2_ 
    from
        company companyent0_ 
    left outer join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.id=?
Hibernate: 
    select
        companyent0_.id as id1_4_1_,
        companyent0_.created_date as created_2_4_1_,
        companyent0_.last_modified_date as last_mod3_4_1_,
        companyent0_.address as address4_4_1_,
        companyent0_.business_registration_number as business5_4_1_,
        companyent0_.name as name6_4_1_,
        companyent0_.phone_number as phone_nu7_4_1_,
        productent1_.company_name as company_7_6_3_,
        productent1_.id as id1_6_3_,
        productent1_.id as id1_6_0_,
        productent1_.created_date as created_2_6_0_,
        productent1_.last_modified_date as last_mod3_6_0_,
        productent1_.company_name as company_7_6_0_,
        productent1_.price as price4_6_0_,
        productent1_.product_name as product_5_6_0_,
        productent1_.stock as stock6_6_0_ 
    from
        company companyent0_ 
    left outer join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.name=?
```

```txt:getCompanyByName()
Hibernate: 
    select
        companyent0_.id as id1_4_,
        companyent0_.created_date as created_2_4_,
        companyent0_.last_modified_date as last_mod3_4_,
        companyent0_.address as address4_4_,
        companyent0_.business_registration_number as business5_4_,
        companyent0_.name as name6_4_,
        companyent0_.phone_number as phone_nu7_4_ 
    from
        company companyent0_ 
    where
        companyent0_.name=?
Hibernate: 
    select
        productent0_.company_name as company_7_6_0_,
        productent0_.id as id1_6_0_,
        productent0_.id as id1_6_1_,
        productent0_.created_date as created_2_6_1_,
        productent0_.last_modified_date as last_mod3_6_1_,
        productent0_.company_name as company_7_6_1_,
        productent0_.price as price4_6_1_,
        productent0_.product_name as product_5_6_1_,
        productent0_.stock as stock6_6_1_ 
    from
        product productent0_ 
    where
        productent0_.company_name=?
Hibernate: 
    select
        companyent0_.id as id1_4_1_,
        companyent0_.created_date as created_2_4_1_,
        companyent0_.last_modified_date as last_mod3_4_1_,
        companyent0_.address as address4_4_1_,
        companyent0_.business_registration_number as business5_4_1_,
        companyent0_.name as name6_4_1_,
        companyent0_.phone_number as phone_nu7_4_1_,
        productent1_.company_name as company_7_6_3_,
        productent1_.id as id1_6_3_,
        productent1_.id as id1_6_0_,
        productent1_.created_date as created_2_6_0_,
        productent1_.last_modified_date as last_mod3_6_0_,
        productent1_.company_name as company_7_6_0_,
        productent1_.price as price4_6_0_,
        productent1_.product_name as product_5_6_0_,
        productent1_.stock as stock6_6_0_ 
    from
        company companyent0_ 
    left outer join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.name=?
```

ID로 조회한 경우 SELECT 쿼리가 총 두 번 발생했고, '다'의 상품 엔티티들을 LEFT OUTER JOIN으로 한 번에 가져온 뒤 FK로 한 번 더 LEFT OUTER JOIN을 수행했습니다. 이와 같이 한 번 더 쿼리가 발생한 이유는 FK를 업체 테이블의 PK인 ID로 설정하지 않아 JPQL이 쿼리를 한 번 더 생성한 것으로 추측됩니다. 이를 증명하려면 PK를 FK로 참조하도록 연관 관계를 설계하여 테스트를 해봐야 할 것 같습니다. ···*(1)*

업체명으로 조회한 경우 SELECT 쿼리가 총 세 번 발생했고, 업체 테이블과 상품 테이블 각각에 대해 업체명을 WHERE 조건으로 SELECT하여 조회한 뒤 FK에 대해 LEFT OUTER JOIN을 수행했습니다. 이 경우 ID로 조회한 경우와 달리 업체 테이블을 조회할 때 상품 테이블로 LEFT OUTER JOIN을 하지 않고 업체, 상품 테이블 각각에 대해 SELECT 쿼리가 발생했습니다. PK인 ID로 조회하는 경우와 아닌 경우 생성되는 JPQL이 다른 것으로 추측됩니다. 그리고 이 때 '다' 엔티티, 즉 상품 테이블에 대해 추가적인 쿼리가 발생하는 문제가 N+1 문제라고 생각됩니다.

#### *(1)* 그래서 별도로 테스트를 진행했습니다
추측으로 끝내기엔 너무 아쉬워서 아예 별도로 테스트 프로젝트를 진행했습니다. 광고 플랫폼 도메인에서 업체-상품 관계와 비슷하게 학과-학생 양방향 일대다 연관 관계에 대해 프로젝트를 설계하고 테스트를 진행했습니다.

<div class="mxgraph" style="max-width:100%; margin:auto;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;lightbox&quot;:false,&quot;nav&quot;:true,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2022-06-20T11:39:38.975Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36\&quot; etag=\&quot;MehTr8dg-lerYRQwnKvO\&quot; version=\&quot;20.0.1\&quot; type=\&quot;google\&quot;&gt;&lt;diagram id=\&quot;Dv1acctghuHOW7tnIxDJ\&quot; name=\&quot;FK PK N+1 Test ERD 1\&quot;&gt;7VpRb9o6FP41PN6JJEDL42D0Xt210kQ37bFysUmsOXbkmAH99TuO7YTghpWOkmpB4iE+OTmxz/f5fPIhvWiabv6VKEvuBCasF/bxphd96oVhMAjDnv718dZYroYDY4glxdapMtzTJ2KNfWtdUUzymqMSgima1Y0LwTlZqJoNSSnWdbelYPW3ZigmnuF+gZhv/U6xSoz1Oryq7P8RGifuzcFobO6kyDnbleQJwmK9Y4pmvWgqhVDmKt1MCdPJc3kxz9003C0nJglXL3lg9v+3wfzh8eNntLqLg+Q7WYT5PzbKT8RWdsGfCIRUqQ5qpq22LhewgkxfKvSoTZNcgaeFLOqDAUBQiHIiwRAUY8ZQltPC3VgSyvAt2oqVcoHcaLKkG4LnBjHtC+DdQjA91MGXEPzeTkbfRozGHK4XMFX9xokkOczlFuXKeti1EanIpjFpQQkFcJiIlCi5BRf3wMiit3W0tON1RYbg2tqSHSKMrQ1Z/sVl6AoiuLAoHYFY6CF2GKe55twkEZI+aXSYzeYudsV4TVOGOJAZ4T3TRBSbt8CAMjYVTGiAueDEw1g7YSmyr0jGRFlDJihXRR6GE/hBZqb9D8PeEOY6hXFQjeGn3aWaCp4rCVzSMQhAuiYa1okSmQ3KyNLFlzbt+vpRKCXSYwjQvC98VlgWRC8kQfRWJIg8Enz53EgDvZ8pYnMoj4jHzIBWVEtUgfYMss/muszvfuL3t6eAtC9ZUfASijHhf4RH+DweOwBER+bfBquycnQ0xKDucKRgg6w4zj1Qy3m+HueBhzO4t4uzK7vGd5JnaEF5fGueHO0RYXguImyaN+bwpMR4UbgzMGN4kYEXyUD/DWVg1LYMjDwSfGtbBgimLl6TCtSV4tXYDLspCVfPSELz3n8nmnBaDWhAvnMacO1R4V6t8OXkVtwdRK88uZXonrxcjy+afYaj2/VBzW796OYaW905u427KdSB31vr+uGtgQmdE+7gUvjf4LB2uPC3flgL/KbdTduF/3yntaCjHbzAb+Hh8h+WB4ofOEqhBL93XTitDjSRoXtCcGnjta8M49aVwe/jvX9d+IPd39HOXeC37qD4k46V/kvzzibC795dSv+5S38Qtl77j2gKvpvaf6ozQQM4f7sQONLtCgFd/HjooBo0MKBzahD6zUKPCQTHxO06SCtV2zlhSFHBZ9UdszeNHMAxCwRDpcxuZcLxR/1JJAxn8ycixVdxh/jW3LmhzEmK1MskpXpogameM6wqjDuPQI0XP8ovJO3L3DT2ZSqoSrpeUyNlrCkXK7kgh5Jn/JSTqka2/b4HUf5LVPtHyBllke+f9RkfIN0XrZc7dHa9f6s+3keCZqX2qYpkXqBBvx4oCvcCmVR4gX7PVhhWX6Qa9+q73mj2Cw==&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
&lt;그림 2. 테스트 프로젝트 ERD - 상황 1: FK가 PK를 참조하는 경우&gt;
{: style="text-align: center;"}

<div class="mxgraph" style="max-width:100%; margin:auto;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;lightbox&quot;:false,&quot;nav&quot;:true,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;app.diagrams.net\&quot; modified=\&quot;2022-06-20T11:41:19.811Z\&quot; agent=\&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36\&quot; etag=\&quot;ezHj5IL3sx-U12QFUxGG\&quot; version=\&quot;20.0.1\&quot; type=\&quot;google\&quot;&gt;&lt;diagram name=\&quot;FK PK N+1 Test ERD 2\&quot; id=\&quot;9oYjQFuCC-lx8KSD_U39\&quot;&gt;7VpRb9owEP41PG4icaDlcVC6SqNSRTftsXKxCdYcO3PMgP762bGdENyU0lJADRIP8eVyie/7fJ98uAUGyfK7gOnsliNMW2EbLVvgqhWGQRSGLf1ro5WxXHQiY4gFQdapNNyTJ2yNbWudE4SziqPknEqSVo0TzhieyIoNCsEXVbcpp9W3pjDGnuF+Aqlv/U2QnBnrZXhR2m8wiWfuzUG3Z+4k0DnbmWQziPhizQSGLTAQnEtzlSwHmOrkubywv+FtZzyEN7/AfDz6Dm+iu8svJtj1Lo8UUxCYyf2GDjsm9j9I5zZhV1i9SCb6VWbacuVyqTKQ6ksJH7Wpn0nlaSEHbWVQIEpIGBbKEORjSmGakdzdWGaEohFc8bl0gdyoPyVLjMYGce2rwB+pYHqog09V8Hv7Mfo2pCRm6nqiPlW/sS9wpr5lBDNpPezcsJB4ucGGLakMCnzVwsA8wVKs1HMuStdSYuW4bseLkmHBpbXN1tjVszZoSR0XoUvg1IXFbhccux6OL6M31kzuz7ggTxozanO8jmg+XpCEQqaWCEQbpj7PS0KODKF0wCnXsDPOsIe8dkKCpz+hiLG0hpQTJvNEdPrqp1IzaH/ttDrqWwdqHJRj9dPuQg44y6RQDNMxsAJ6gTXYfclTG5TiqYsvbN719SOXkifvpoVZLj4vLA/AK2kAPowGFx4N7n7UEkGvcwLpWJVdyGJqYMurMCxhewbbZ7NdZHgz9ZvLlqvET2leSGcEIczeh0j3eUTWIAA7ImCDlWnZORqkqiAxKNUamTOUebAW3/kOpC89pJX/cZF2Bdn49rMUTgiLR+bJ7gYVOgejwrJ+cXb2So1XhTsEN3pnMXiVGLQ/Ugy6xxYDF3iNBr+OLQYYERevTguqevF2dHrNFAYQeKgTVL/+T0QZ9qwENdg3TglA6JHhXs7ReW9XT44IvHFvV2C+/0IOznp+gM2dWSynu7kDkUeDT765M8RvoIb7Xbmmb+7qqNA8Se+ea//+93Jbav/x93J+Y+/62LX/cHs50NAmH/CbfKj4d+aBoAcGE1WET10Z9qwE5zafTcS5zXcC0tA7tjREfpvv9IXhHXA0tK8X+X09Vf1xw2r/ubFn2eA39s61/+C1PwiPXvx3aA2eTPHf064gqgHn0yuB3wdkZPLnoXlyUMeA5smB3zD0mIBRjN2qU2klcjXGFErC2bC8Y9am0YMg1IohE2qXMmbomz6QqYbD8RMW/Ce/hWxl7lwT6jRF6GniQj60wpTPGVblxrVHVJHnf4rzmfZl7jM2dSooa7qe026UsX4Zn4sJfsHPnaGQTsFqJWJ7b6L4C6nyd5EzihyFf9V5vEDFOy2jayR3B22tJnlnDM1U7VMl9bxAUbsaCIQbgUwqvEDbOayG5SlZ416eNQbD/w==&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>
&lt;그림 3. 테스트 프로젝트 ERD - 상황 2: FK가 PK가 아닌 필드를 참조하는 경우&gt;
{: style="text-align: center;"}

우선 1번 상황에서, '일' 엔티티의 PK로 조회하는 테스트를 수행했습니다.

<p class=short>엔티티</p>

```java
@Getter
@Setter
@NoArgsConstructor(access=AccessLevel.PROTECTED)
@Entity
@Table(name="DEPARTMENT")
public class DepartmentEntity implements Serializable {
    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    @Column(nullable=false, unique=true)
    private Long id2;

    @OneToMany(mappedBy="departmentEntity", fetch=FetchType.EAGER)
    private List<StudentEntity> studentEntities = new ArrayList<>();

    @Builder
    public DepartmentEntity(Long id, Long id2, List<StudentEntity> studentEntities) {
        this.id = id;
        this.id2 = id2;
        this.studentEntities = studentEntities;
    }

    public DepartmentDto toDto() {
        return DepartmentDto.builder()
                .id(this.getId())
                .id2(this.getId2())
                .studentDtos(this.getStudentEntities().stream().map(ent -> ent.toDtoExceptDepartment()).collect(Collectors.toList()))
                .build();
    }

    public DepartmentDto toDtoExceptStudents() {
        return DepartmentDto.builder()
                .id(this.getId())
                .id2(this.getId2())
                .build();
    }

    public void addStudentEntity(StudentEntity studentEntity) {
        if (this.getStudentEntities() == null) {
            this.setStudentEntities(new ArrayList<>());
        }

        this.studentEntities.add(studentEntity);
    }
}

@Getter
@Setter
@NoArgsConstructor(access=AccessLevel.PROTECTED)
@Entity
@Table(name="STUDENT")
public class StudentEntity {
    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    @Column(nullable=false)
    private String name;

    @Column(nullable=false, unique=true)
    private String nickName;

    @ManyToOne
    @JoinColumn(name="department_id_naming", referencedColumnName="id")
    private DepartmentEntity departmentEntity;

    @Builder
    public StudentEntity(Long id, String name, String nickName, DepartmentEntity departmentEntity) {
        this.id = id;
        this.name = name;
        this.nickName = nickName;
        this.departmentEntity = departmentEntity;
    }

    public StudentDto toDto() {
        return StudentDto.builder()
                .id(this.getId())
                .name(this.getName())
                .departmentDto(this.getDepartmentEntity().toDtoExceptStudents())
                .build();
    }

    public StudentDto toDtoExceptDepartment() {
        return StudentDto.builder()
                .id(this.getId())
                .name(this.getName())
                .build();
    }
}
```

<p class=short>테스트 코드(1번 상황, FK -> PK, PK로 조회)</p>

```java
@SpringBootTest
class TestApplicationTests {
    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private DepartmentRepository departmentRepository;

    @Test
    void createDepartmentAndStudent() {
        DepartmentDto departmentDto = DepartmentDto.builder()
                .id2(100001L)
                .studentDtos(new ArrayList<>())
                .build();

        DepartmentEntity createdDepartmentEntity = departmentRepository.save(departmentDto.toEntityExceptStudents());

        List<StudentDto> studentDtos = new ArrayList<>();

        studentDtos.add(StudentDto.builder()
                .name("testName1")
                .nickName("testNickName1")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName2")
                .nickName("testNickName2")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName3")
                .nickName("testNickName3")
                .build());

        for (StudentDto studentDto: studentDtos) {
            StudentEntity studentEntity = studentDto.toEntityExceptDepartment();
            studentEntity.setDepartmentEntity(createdDepartmentEntity);
            studentRepository.save(studentEntity);
        }

        System.out.println("After inserts,");

        DepartmentEntity resDepartmentEntity = departmentRepository.findById(1L).get();

        int i = 1;
        for (StudentEntity resStudentEntity: resDepartmentEntity.getStudentEntities()) {
            Assertions.assertEquals(resStudentEntity.getNickName(), "testNickName"+i++);
        }
    }
}
```

<p class=short>결과(1번 상황, FK -> PK, PK로 조회)</p>

```txt
After inserts,
Hibernate: 
    select
        department0_.id as id1_0_0_,
        department0_.id2 as id2_0_0_,
        studentent1_.department_id_naming as departme4_1_1_,
        studentent1_.id as id1_1_1_,
        studentent1_.id as id1_1_2_,
        studentent1_.department_id_naming as departme4_1_2_,
        studentent1_.name as name2_1_2_,
        studentent1_.nick_name as nick_nam3_1_2_ 
    from
        department department0_ 
    left outer join
        student studentent1_ 
            on department0_.id=studentent1_.department_id_naming 
    where
        department0_.id=?
```

**PK를 FK로 참조하는 관계에서 '일' 엔티티에 대해 PK로 조회하면 SELECT 쿼리가 한 개만 발생합니다.** 제 추측이 맞아들어가고 있다는 증거입니다. 다음으로, 1번 상황에서 '일' 엔티티에 대해 PK가 아닌 다른 필드 `id2`로 조회하는 테스트 코드를 수행했습니다.

<p class=short>테스트 코드(1번 상황, FK -> PK, !PK로 조회)</p>

```java
@SpringBootTest
class TestApplicationTests {
    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private DepartmentRepository departmentRepository;

    @Test
    void createDepartmentAndStudent() {
        DepartmentDto departmentDto = DepartmentDto.builder()
                .id2(100001L)
                .studentDtos(new ArrayList<>())
                .build();

        DepartmentEntity createdDepartmentEntity = departmentRepository.save(departmentDto.toEntityExceptStudents());

        List<StudentDto> studentDtos = new ArrayList<>();

        studentDtos.add(StudentDto.builder()
                .name("testName1")
                .nickName("testNickName1")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName2")
                .nickName("testNickName2")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName3")
                .nickName("testNickName3")
                .build());

        for (StudentDto studentDto: studentDtos) {
            StudentEntity studentEntity = studentDto.toEntityExceptDepartment();
            studentEntity.setDepartmentEntity(createdDepartmentEntity);
            studentRepository.save(studentEntity);
        }

        System.out.println("After inserts,");

        DepartmentEntity resDepartmentEntity = departmentRepository.findById2(100001L).get();

        int i = 1;
        for (StudentEntity resStudentEntity: resDepartmentEntity.getStudentEntities()) {
            Assertions.assertEquals(resStudentEntity.getNickName(), "testNickName"+i++);
        }
    }
}
```

<p class=short>결과(1번 상황, FK -> PK, !PK로 조회)</p>

```txt
After inserts,
Hibernate: 
    select
        department0_.id as id1_0_,
        department0_.id2 as id2_0_ 
    from
        department department0_ 
    where
        department0_.id2=?
Hibernate: 
    select
        studentent0_.department_id_naming as departme4_1_0_,
        studentent0_.id as id1_1_0_,
        studentent0_.id as id1_1_1_,
        studentent0_.department_id_naming as departme4_1_1_,
        studentent0_.name as name2_1_1_,
        studentent0_.nick_name as nick_nam3_1_1_ 
    from
        student studentent0_ 
    where
        studentent0_.department_id_naming=?
```

비록 FK로 PK를 참조하고 있더라도 '일' 엔티티에 대해 PK로 조회하지 않는다면 '다' 엔티티에서 FK를 WHERE 조건으로 SELECT 쿼리가 추가로 발생합니다.

마지막으로 광고 플랫폼 도메인 상황과 같은 2번 상황으로 연관 관계를 설계하고 PK 필드로 조회, PK 필드가 아닌 경우로 조회하는 테스트를 진행했는데, 광고 플랫폼에서 발생했던 쿼리와 동일했습니다. 각각 2, 3개의 SELECT 쿼리가 발생했으며 마지막 SELECT에는 LEFT OUTER JOIN 절이 보이는 점까지 동일했는데, 이 마지막 SELECT문은 '다' 엔티티의 FK가 '일' 엔티티의 PK를 참조할 때 사용한 조회 쿼리에서 추가된 모습이었습니다. '다' 엔티티의 FK가 PK를 참조하지 않으니 앞선 쿼리에서 조회한 데이터들을 무시하고 다시 LEFT OUTER JOIN으로 WHERE 조건에 FK를 사용하여 가져오는 느낌입니다.

후술하겠지만, N+1 문제를 해결하는 방법으로 Fetch Join을 가장 많이 사용합니다. 이 때 1과 같이 일대다 관계에서 '다' 엔티티의 FK가 '일' 엔티티의 PK를 참조하는 경우 Fetch Join을 사용했을 때 결과적으로 SELECT 쿼리가 **한 번만** 발생합니다. 이에 대해서도 직접 테스트해봤습니다.

<p class=short>테스트 코드(1번 상황 FK -> PK, Fetch Join, findAll())</p>

```java
public interface DepartmentRepository extends JpaRepository<DepartmentEntity, Long> {
    @Query("SELECT DISTINCT d FROM DepartmentEntity d JOIN FETCH d.studentEntities")
    List<DepartmentEntity> findAllFetch();
}

@SpringBootTest
class TestApplicationTests {
    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private DepartmentRepository departmentRepository;

    @Test
    void createDepartmentAndStudent() {
        DepartmentDto departmentDto = DepartmentDto.builder()
                .id2(100001L)
                .studentDtos(new ArrayList<>())
                .build();

        DepartmentEntity createdDepartmentEntity = departmentRepository.save(departmentDto.toEntityExceptStudents());

        List<StudentDto> studentDtos = new ArrayList<>();

        studentDtos.add(StudentDto.builder()
                .name("testName1")
                .nickName("testNickName1")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName2")
                .nickName("testNickName2")
                .build());
        studentDtos.add(StudentDto.builder()
                .name("testName3")
                .nickName("testNickName3")
                .build());

        for (StudentDto studentDto: studentDtos) {
            StudentEntity studentEntity = studentDto.toEntityExceptDepartment();
            studentEntity.setDepartmentEntity(createdDepartmentEntity);
            studentRepository.save(studentEntity);
        }

        System.out.println("After inserts,");

        List<DepartmentEntity> resDepartmentEntities = departmentRepository.findAllFetch();

        int i = 1;
        for (StudentEntity resStudentEntity: resDepartmentEntities.get(0).getStudentEntities()) {
            Assertions.assertEquals(resStudentEntity.getNickName(), "testNickName"+i++);
        }
    }
}
```

<p class=short>결과(1번 상황 FK -> PK, Fetch Join, findAll())</p>

```txt
After inserts,
Hibernate: 
    select
        distinct department0_.id as id1_0_0_,
        studentent1_.id as id1_1_1_,
        department0_.id2 as id2_0_0_,
        studentent1_.department_id_naming as departme4_1_1_,
        studentent1_.name as name2_1_1_,
        studentent1_.nick_name as nick_nam3_1_1_,
        studentent1_.department_id_naming as departme4_1_0__,
        studentent1_.id as id1_1_0__ 
    from
        department department0_ 
    inner join
        student studentent1_ 
            on department0_.id=studentent1_.department_id_naming
```

#### 그래서 결론은
많이 헤맸는데 결과적으로 결론은 다음과 같습니다.

1. '다' 엔티티의 FK가 '일' 엔티티의 PK를 참조하는 경우(FK -> PK, '일'.FK참조필드=PK필드)
    - '일' 엔티티의 PK로 조회하는 경우 **N+1 문제가 발생하지 않음**
        - SELECT '일', '다' FROM '일' LEFT OUTER JOIN '다' ON FK='일'.FK참조필드 WHERE '일'.PK필드=? (1번)
    - '일' 엔티티의 PK가 아닌 필드로 조회하는 경우 **N+1 문제 발생**
        - SELECT '일' WHERE 임의필드=? (1번)
        - SELECT '다' WHERE FK='일'.FK참조필드값 (1번)
    - '일' 엔티티를 findAll()로 전체 조회하는 경우 **N+1 문제 발생**
        - SELECT '일' (1번) (전체 조회)
        - SELECT '다' WHERE FK='일'.FK참조필드값 ('일' 엔티티 개수 K개 각각에 대해 발생, K번)
2. '다' 엔티티의 FK가 '일' 엔티티의 PK가 아닌 다른 필드를 참조하는 경우(FK -> !PK, '일'.FK참조필드≠PK필드)
    - 1 하위 항목의 각 조회 상황과 동일할 때 결과 쿼리에 다음 쿼리가 추가됨
        - SELECT '일', '다' FROM '일' LEFT OUTER JOIN '다' ON FK='일'.FK참조필드 WHERE '일'.FK참조필드='일'.FK참조필드값 ('일' 엔티티 개수 K개 각각에 대해 발생, K번)
    - '다' 엔티티의 FK가 PK를 참조하지 않으니 앞선 쿼리에서 조회한 '일' 데이터의 FK참조필드값을 WHERE 조건에 사용하여 LEFT OUTER JOIN으로 다시 가져오는 느낌으로 **N+1 문제가 아님**
3. 위 모든 경우에 대해 Fetch Join을 사용하면, 명시한 N+1 문제 상황에 대해 해결됨

FK로 PK를 참조하지 않는 것이 추가 SELECT - LEFT OUTER JOIN문을 발생시키는 것으로 보입니다. 그런데 **기존에 알려져 있는 N+1 문제처럼 '다' 엔티티의 개수 N개의 추가 SELECT 쿼리가 발생하진 않았습니다.** '일'의 엔티티 개수 K개 만큼만 '다' 엔티티에 대한 SELECT문이 발생했다는 것이 차이점입니다. 제가 감히 재명명을 해보자면 1+1 문제('일' 엔티티 조회 1건에 대해서 추가 1건의 SELECT문 발생)가 될 것 같습니다.

2번에 대해서 고찰을 해봤는데, 2번과 같은 상황이 벌어지는 이유를 짐작하기가 어려웠습니다. 추측해보자면 혹시나 유일성을 보장하지 못하는 필드를 FK로 참조하고 있는 경우를 위한 조치일 수도 있고, 또는 예전의 N+1 문제의 원인이었던 것처럼 SQL 생성 시 엔티티 간 참조 관계를 고려하지 않은 것일 수도 있고 이 부분은 JPA 구현체 Hibernate의 코드를 전부 뜯어봐야 알 것 같습니다..

#### 다시 돌아와서
1+1 문제는 여전히 남아있으므로, N+1 문제에 대해서 조사하다보니 [Hibernate 공식 가이드 문서](https://docs.jboss.org/hibernate/orm/6.0/userguide/html_single/Hibernate_User_Guide.html#fetching)에서 Fetch Join으로 문제를 해결할 수 있다는 것을 확인했습니다. JPQL을 `@Query` 어노테이션을 통해 직접 작성하고 이 때 Fetch Join 구문을 활용하는 방법입니다. 제가 겪는 문제가 예전부터 이슈가 됐던 N+1문제랑은 조금 다르지만 쿼리문에 변화가 생기는지, 결과적으로 성능 향상이 기대되는지 확인하기 위해 Fetch Join을 적용했습니다.

```java
/**
 * 업체 도메인의 비즈니스 로직을 처리하는 서비스 클래스
 */
@Service
@RequiredArgsConstructor
public class CompanyService {
    private final ProductRepository productRepository;
    private final CompanyRepository companyRepository;

    @Transactional(readOnly=true)
    public CompanyDto findById(Long id) {
        Optional<CompanyEntity> companyEntityWrapper = companyRepository.findByIdJpqlFetch(id);

        if (companyEntityWrapper.isPresent()) {
            return companyEntityWrapper.get().toDto();
        }

        return null;
    }

    @Transactional
    public CompanyDto findByName(String companyName) {
        Optional<CompanyEntity> companyEntityWrapper = companyRepository.findByNameJpqlFetch(companyName);

        if (companyEntityWrapper.isPresent()) {
            return companyEntityWrapper.get().toDto();
        }

        return null;
    }
}

/**
 * CompanyEntity의 영속성을 관리하며 엔티티에 매핑된 DB 테이블 'COMPANY'에 CRUD 기능 수행
 * JOIN FETCH로 쿼리 성능 향상
 */
public interface CompanyRepository extends JpaRepository<CompanyEntity, Long> {
    @Query("SELECT DISTINCT e FROM CompanyEntity e JOIN FETCH e.productEntities WHERE e.id=:id")
    Optional<CompanyEntity> findByIdJpqlFetch(@Param("id") Long id);

    @Query("SELECT DISTINCT e FROM CompanyEntity e JOIN FETCH e.productEntities WHERE e.name=:name")
    Optional<CompanyEntity> findByNameJpqlFetch(@Param("name") String name);
}
```

![](https://drive.google.com/uc?export=view&id=1xovzuk7kZgQQGq-RL65AoBuZzzGxywvP){: .align-center}
&lt;화면 3. getCompanyById(), getCompanyByName() API 테스트 결과(with fetch)&gt;
{: style="text-align: center;"}

소요 시간을 비교해보니, 단일 레코드를 조회하는데 Fetch Join으로 유의미한 성능 향상을 기대하진 못할 것 같습니다. 다음으로 쿼리를 살펴봤습니다.

```txt:getCompanyByIdWithFetch()
Hibernate: 
    select
        distinct companyent0_.id as id1_4_0_,
        productent1_.id as id1_6_1_,
        companyent0_.created_date as created_2_4_0_,
        companyent0_.last_modified_date as last_mod3_4_0_,
        companyent0_.address as address4_4_0_,
        companyent0_.business_registration_number as business5_4_0_,
        companyent0_.name as name6_4_0_,
        companyent0_.phone_number as phone_nu7_4_0_,
        productent1_.created_date as created_2_6_1_,
        productent1_.last_modified_date as last_mod3_6_1_,
        productent1_.company_name as company_7_6_1_,
        productent1_.price as price4_6_1_,
        productent1_.product_name as product_5_6_1_,
        productent1_.stock as stock6_6_1_,
        productent1_.company_name as company_7_6_0__,
        productent1_.id as id1_6_0__ 
    from
        company companyent0_ 
    inner join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.id=?
Hibernate: 
    select
        companyent0_.id as id1_4_1_,
        companyent0_.created_date as created_2_4_1_,
        companyent0_.last_modified_date as last_mod3_4_1_,
        companyent0_.address as address4_4_1_,
        companyent0_.business_registration_number as business5_4_1_,
        companyent0_.name as name6_4_1_,
        companyent0_.phone_number as phone_nu7_4_1_,
        productent1_.company_name as company_7_6_3_,
        productent1_.id as id1_6_3_,
        productent1_.id as id1_6_0_,
        productent1_.created_date as created_2_6_0_,
        productent1_.last_modified_date as last_mod3_6_0_,
        productent1_.company_name as company_7_6_0_,
        productent1_.price as price4_6_0_,
        productent1_.product_name as product_5_6_0_,
        productent1_.stock as stock6_6_0_ 
    from
        company companyent0_ 
    left outer join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.name=?
```

```txt:getCompanyByNameWithFetch()
Hibernate: 
    select
        distinct companyent0_.id as id1_4_0_,
        productent1_.id as id1_6_1_,
        companyent0_.created_date as created_2_4_0_,
        companyent0_.last_modified_date as last_mod3_4_0_,
        companyent0_.address as address4_4_0_,
        companyent0_.business_registration_number as business5_4_0_,
        companyent0_.name as name6_4_0_,
        companyent0_.phone_number as phone_nu7_4_0_,
        productent1_.created_date as created_2_6_1_,
        productent1_.last_modified_date as last_mod3_6_1_,
        productent1_.company_name as company_7_6_1_,
        productent1_.price as price4_6_1_,
        productent1_.product_name as product_5_6_1_,
        productent1_.stock as stock6_6_1_,
        productent1_.company_name as company_7_6_0__,
        productent1_.id as id1_6_0__ 
    from
        company companyent0_ 
    inner join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.name=?
Hibernate: 
    select
        companyent0_.id as id1_4_1_,
        companyent0_.created_date as created_2_4_1_,
        companyent0_.last_modified_date as last_mod3_4_1_,
        companyent0_.address as address4_4_1_,
        companyent0_.business_registration_number as business5_4_1_,
        companyent0_.name as name6_4_1_,
        companyent0_.phone_number as phone_nu7_4_1_,
        productent1_.company_name as company_7_6_3_,
        productent1_.id as id1_6_3_,
        productent1_.id as id1_6_0_,
        productent1_.created_date as created_2_6_0_,
        productent1_.last_modified_date as last_mod3_6_0_,
        productent1_.company_name as company_7_6_0_,
        productent1_.price as price4_6_0_,
        productent1_.product_name as product_5_6_0_,
        productent1_.stock as stock6_6_0_ 
    from
        company companyent0_ 
    left outer join
        product productent1_ 
            on companyent0_.name=productent1_.company_name 
    where
        companyent0_.name=?

```

ID로 조회하는 쿼리는 여전히 2개의 SELECT 쿼리가 발생했는데, Fetch Join으로 LEFT OUTER JOIN 대신 INNER JOIN을 수행한 것으로 보입니다. 업체명으로 조회하는 쿼리는 기존 세 개의 SELECT 쿼리에서 두 개로 줄었는데 이는 '다' 엔티티에 대해 FK를 WHERE 조건으로 한 SELECT문이 Fetch Join으로 없어진 모습입니다. 결과적으로 1+1 문제가 해결된 모습인데, 단일 레코드 조회가 아니라 전체 리스트 조회 시 쿼리가 1/2로 줄어드는 것이므로 성능 향상이 있을 것으로 기대됩니다.

그리고 현재 광고 플랫폼 도메인의 설계와 달리, 일대다 관계에서 '다' 엔티티가 FK로 '일' 엔티티의 PK를 참조한다면 리스트 전체 조회에 Fetch Join문을 사용했을 때 **쿼리가 단 한개**만 발생합니다. 따라서 FK가 반드시 PK를 참조하도록 설계해야 JPA의 성능이 극대화될 수 있음을 알 수 있습니다.

## 배치 처리에 필요한 복잡한 쿼리와 Native Query, QueryDSL
비즈니스 로직에 대한 코딩도 끝나갈 무렵 마지막 로직인 정산 도메인의 일별 집계 데이터 생성에서 JPA를 어떻게 작성해야 하는지 고민했었습니다. 정산 집계한 데이터는 여러 테이블의 필드를 참조해야 했으며 집계 함수를 적용한 값을 필드 값으로서 사용해야 한다는 점이 JPA를 작성하는데 어려움을 주었습니다. 그리고 이런 문제는 JPA 사용자들의 공통적인 어려움이어서, 비즈니스 로직에 이런 복잡한 쿼리가 필요할 때는 JPQL, 또는 SQL을 직접 작성할 수 있는 Native Query를 이용한다는 것을 알게 됐습니다.

일별 광고 정산 시 생성되는 집계 데이터에는 클릭일자, 업체 ID, 업체명, 광고입찰 ID, 상품 ID, 상품명, 클릭수, 총과금액 을 가지고 있어야 하고, 총과금액은 클릭일자, 광고입찰 ID 별 클릭수 x 광고입찰가 이어야 한다는 요구사항이 있었습니다. 저는 이를 보고 (클릭일자, 광고입찰 ID) 두 개의 필드를 테이블의 기본 키로 적용해야 한다는 의미로 생각했고(즉, 복합키), 필요한 필드를 살펴봤을 때 광고과금 테이블의 FK(광고입찰 ID)로 광고입찰 테이블을 참조하고, 광고입찰 테이블의 FK2(상품 ID)로 상품 테이블을 참조하고, 상품 테이블의 FK(업체명)로 업체 테이블을 참조하여 업체 ID, 업체명, 상품 ID, 상품명을 알아내고 이를 엔티티로 담아내야 한다고 결정했습니다.

<p class=short>AdChargeCalEntity</p>

```java
/**
 * JPA에서 엔티티의 복합키를 정의하는 클래스
 */
@Data
@NoArgsConstructor(access=AccessLevel.PROTECTED)
@Embeddable
public class AdChargeCalId implements Serializable {
    @Column(name="clicked_date")
    private LocalDate clickedDate;

    @Column(name="bid_id")
    private Long bidId;

    @Builder
    public AdChargeCalId(LocalDate clickedDate, Long bidId) {
        this.clickedDate = clickedDate;
        this.bidId = bidId;
    }
}

/**
 * DB 테이블 'ADVERTISEMENT_CHARGE_CAL'에 매핑되는 엔티티 클래스
 * 클릭일자, 광고입찰 ID 두 개 컬럼을 복합키로 가짐
 */
@Getter
@Setter
@NoArgsConstructor(access=AccessLevel.PROTECTED)
@Entity
@Table(name="ADVERTISEMENT_CHARGE_CAL")
public class AdChargeCalEntity {
    @EmbeddedId
    private AdChargeCalId adChargeCalId;

    @Column
    private Long companyId;

    @Column
    private String companyName;

    @Column
    private Long productId;

    @Column
    private String productName;

    @Column
    private Long cntClicked;

    @Column
    private Long totalCharge;
    ...
}
```

그리고 수행해야 될 쿼리를 생각해보면, 일별 정산이면서 클릭 횟수 등을 집계해야 하기 때문에 GROUP BY와 집계 함수 COUNT를 사용해야 함을 알 수 있습니다. 여기서 작일 날짜를 조건으로 사용해야 하는데, GROUP BY의 HAVING 절에 사용할지 WHERE 절에 사용할지 인라인 뷰(inline-view)로 사용할지 다양한 선택지가 있었습니다. 저는 인라인 뷰가 테이블 조인 전에 조건을 검사하면서 쿼리의 로직도 단순화시킬 수 있는 좋은 선택지일 것이란 생각이 들었는데, 문득 HAVING 절에 사용하는 것과 인라인 뷰/WHERE 절을 사용하는 것에 성능 차이가 있는지 궁금했습니다. 조사해보니 SQL 표준 이론에 의하면 WHERE 절은 테이블 집계 전 결과 집합을 반환하기 전에 조건을 적용하는 것이고, HAVING 절은 결과 집합을 반환받은 전체 행에 조건을 적용하는 것으로 WHERE 절을 사용하는 것이 성능에 유리하며, 가급적 WHERE 절을 사용할 수 없는 상황에서 HAVING 절을 사용하는 것이 좋다고 합니다([stackoverflow, 2008.](https://stackoverflow.com/questions/328636/which-sql-statement-is-faster-having-vs-where)).

결과적으로 다음과 같은 SQL 쿼리문을 사용해서 광고 플랫폼에서의 광고과금 일별 정산 집계 데이터를 얻을 수 있었습니다. H2 데이터베이스의 SQL 문법이 MySQL 문법과 다소 달라 [H2 공식 문서 - SQL 문법](https://www.h2database.com/html/grammar.html)을 참조했습니다.

```sql
SELECT AC.CLICKED_DATE, AC.BID_ID, C.ID AS COMPANY_ID, C.NAME AS COMPANY_NAME, P.ID AS PRODUCT_ID, P.PRODUCT_NAME, COUNT(*) AS CNT_CLICKED, COUNT(*)*AB.BID_PRICE AS TOTAL_CHARGE
FROM (SELECT ID, FORMATDATETIME(CLICKED_DATE, 'yyyy-MM-dd') AS CLICKED_DATE, BID_ID, BID_PRICE FROM ADVERTISEMENT_CHARGE WHERE FORMATDATETIME(CLICKED_DATE, 'yyyy-MM-dd')=DATEADD('DAY', -1, CURRENT_DATE)) AC, ADVERTISEMENT_BID AB, COMPANY C, PRODUCT P
WHERE AC.BID_ID = AB.ID AND AB.COMPANY_ID = C.ID AND AB.PRODUCT_ID = P.ID
GROUP BY CLICKED_DATE, BID_ID
```

<p class=short>Native Query를 사용하는 AdChargeCalRepository</p>

```java
/**
 * AdChargeCalEntity의 영속성을 관리하며 엔티티에 매핑된 DB 테이블 'ADVERTISEMENT_CHARGE_CAL'에 CRUD 기능 수행
 * 복잡한 집계 쿼리를 처리하기 위한 Native Query 사용
 */
public interface AdChargeCalRepository extends JpaRepository<AdChargeCalEntity, AdChargeCalId> {
    @Query(value="SELECT AC.CLICKED_DATE, AC.BID_ID, C.ID AS COMPANY_ID, C.NAME AS COMPANY_NAME, P.ID AS PRODUCT_ID, P.PRODUCT_NAME, COUNT(*) AS CNT_CLICKED, COUNT(*)*AB.BID_PRICE AS TOTAL_CHARGE " +
            "FROM (SELECT ID, FORMATDATETIME(CLICKED_DATE, 'yyyy-MM-dd') AS CLICKED_DATE, BID_ID, BID_PRICE FROM ADVERTISEMENT_CHARGE WHERE FORMATDATETIME(CLICKED_DATE, 'yyyy-MM-dd')=DATEADD('DAY', -1, CURRENT_DATE)) AC, " +
            "ADVERTISEMENT_BID AB, COMPANY C, PRODUCT P " +
            "WHERE AC.BID_ID = AB.ID AND AB.COMPANY_ID = C.ID AND AB.PRODUCT_ID = P.ID " +
            "GROUP BY CLICKED_DATE, BID_ID"
            , nativeQuery=true)
    public List<AdChargeCalEntity> dailyCal();
}
```

이후에 별다른 수정이 없었는데, 당시에 인지하고 있었던 Native Query의 문제점이 있었습니다. JPQL을 직접 작성하거나 이와 같이 Native Query의 SQL을 직접 작성했을 때 겪었던 문제점으로, 쿼리가 전부 문자열로 작성되기 때문에 Type-safe 하지 않아 컴파일 과정에서 오류를 발견하지 못하고 런타임 환경에서 코드가 실행된 뒤에 오류를 발견할 수 있다는 것이었습니다. 특히 본문에서 다룬 위 쿼리처럼 쿼리가 복잡하고 길 수록 오타가 있거나 문법 오류 때문에 이런 문제를 겪을 가능성이 큽니다.

과제를 끝낸 뒤 위 문제에 대해서 고민하고 조사하다가, JPQL과 Native Query가 Type-safe 하지 못하다는 점을 해결할 수 있는 방법으로 QueryDSL을 사용하는 방법이 있다는 것을 알게됐습니다. 

### QueryDSL
작성 예정

## TDD, 그리고 Spring REST Docs
Spring 어플리케이션의 REST API를 문서화할 때 보통 Swagger와 REST Docs를 사용한다고 알고 있습니다. 과제 전달을 받았을 때도 둘 중 하나를 선택하여 개발한 API를 문서화하도록 했는데 저는 결과적으로 REST Docs를 선택했습니다.

<p class=short>Swagger는 마치 Postman처럼 API를 직접 요청하고 테스트하고 이를 볼 수 있는 화면을 제공하여 동작 테스트하는 용도에 특화되어 있습니다. 이런 장점이 있음에도 불구하고 다음과 같은 단점을 가지고 있습니다.</p>

- 프로덕션 코드, 즉 비즈니스 로직에 어노테이션을 추가하여 명세를 작성함
    - 어플리케이션 동작과는 무관한 코드들이 추가되어 가독성이 떨어짐
- 테스트 기반이 아니기 때문에 명세된 문서가 100% 정확하다고 확신할 수 없음
- 모든 오류에 대한 여러 가지 응답을 문서화할 수 없음

<p class=short>반면 REST Docs는 다음과 같은 장점을 가집니다.</p>

- 테스트 기반으로 문서가 작성되고, 테스트에 실패하면 문서가 작성되지 않기 때문에 작성된 문서에 대한 신뢰 가능
- 테스트 코드에서 명세를 작성하기 때문에 비즈니스 로직의 가독성에 영향을 미치지 않음

또한 본 프로젝트에서는 비즈니스와 관련된 복잡한 유효성 검증 기능을 구현해야 했기 때문에 Spring REST Docs를 이용해 API 문서를 작성했습니다.

그리고 Spring REST Docs를 이용해 개발을 진행하니 자연스럽게 TDD 방식으로 개발을 진행할 수 있었고, 프로젝트 개발 기간 2주라는 비교적 짧은 기간에 목표했던 기능들을 빠르고 정확하게 개발할 수 있었던 원동력이 되었습니다.

<p class=short>다음 코드는 업체 등록 시 유효하지 않은 입력 값(ex. 등록되지 않은 업체 이름, 잘못된 형식의 사업자 번호 등)에 대해 404 결과를 응답하는 API를 테스트하는 코드입니다.</p>

```java:/src/test/java/.../ApiTests
@SpringBootTest
@TestMethodOrder(MethodOrderer.DisplayName.class)
@AutoConfigureMockMvc
@AutoConfigureRestDocs
@Import(RestDocsConfig.class)
public class ApiTests {
    @Autowired
    private MockMvc mockMvc;

    private ObjectMapper objectMapper = new ObjectMapper();

    ...

    @Test
    @DisplayName("05. 업체 등록 시 유효성 검사")
    void registerCompanyValidation() throws Exception {
        CompanyReqDto companyReqDto = CompanyReqDto.builder()
                .companyName("등록되지 않은 업체 이름")
                .businessRegistrationNumber("123456-78901")
                .phoneNumber("010-1234567")
                .build();

        this.mockMvc.perform(
                        post("/api/company")
                                .contentType(MediaType.APPLICATION_JSON)
                                .content(objectMapper.writeValueAsString(companyReqDto))
                )
                .andExpect(status().isBadRequest())
                .andDo(
                        document("POST_api-company-validation",
                                requestFields(
                                        fieldWithPath("companyName").description("Company name which is not appeared in product list"),
                                        fieldWithPath("businessRegistrationNumber").description("Company's business registration number"),
                                        fieldWithPath("phoneNumber").description("Company's phone number"),
                                        fieldWithPath("address").description("Company's address")
                                ),
                                responseFields(
                                        fieldWithPath("timestamp").description("API requested time"),
                                        fieldWithPath("code").description("HTTP status code"),
                                        fieldWithPath("status").description("HTTP status"),
                                        fieldWithPath("result").description("An array of validation result"),
                                        fieldWithPath("result.[].field").description("The field which is invalid"),
                                        fieldWithPath("result.[].message").description("Description message"),
                                        fieldWithPath("result.[].rejectedValue").description("The value rejected").optional()
                                )
                        )
                );
    }

    ...
}
```

위와 같이 Spring Boot Test의 MockMvc 객체를 이용한 테스트 코드와 함께 REST Docs를 작성하도록 했습니다. 기능 개발에 앞서 해당 API를 통해 클라이언트가 반환받아야 할 응답이 어떤 식일지 고민하여 테스트 코드를 작성하고 이후 실제 기능을 개발했습니다.

![](https://drive.google.com/uc?export=view&id=1KDVyXdHyFgLY6-GBZiM6OLDfbgsa494x){: .align-center}
&lt;화면 4. 화면 4. REST Docs - mockMvc 테스트 결과&gt;
{: style="text-align: center;"}

기능을 개발하고 테스트할 때 &lt;화면 4&gt;와 같이 새로 추가한 기능이 잘 동작하는지, 또한 기존에 작성된 기능이 여전히 잘 동작하는지 회귀 테스트를 진행했습니다.

한 편 Spring REST Docs와 함께 TDD로 개발을 하면서 스프링 부트가 제공하는 테스트 종류가 정말 다양함을 알게 됐습니다. @SpringBootTest 어노테이션부터 Assertion 모듈, Mockito 객체를 활용한 테스트 방법과 도커 컨테이너를 테스트 환경에서 실행시킬 수 있는 라이브러리 Testcontainers 등 나중에 이들에 대한 내용을 정리할 기회를 가져야 할 듯 합니다.

## 비즈니스와 관련된 복잡한 유효성 검증
본 프로젝트에서 눈여겨 보았던 것들 중 하나가 데이터 유효성 검증에 대한 요구사항이었습니다. 도메인 종류가 적지 않은 상황에서 비즈니스 로직에 따른 데이터 관계에 대한 특별한 유효성 검증이 필요했는데, 요구사항에서 이런 특별한 유효성 검증이 필요한 부분을 살펴보면 다음과 같았습니다.

- 업체
    - 상품에 세팅된 업체만 등록 가능(최초 등록 시 업체명 기준 체크)
        - 등록 후 상품정보와의 연계성을 가진 업체명을 제외한 업체정보만 업데이트 가능
    - 사업자번호, 업체전화번호 등의 데이터 세팅 시 자릿수, 숫자 여부 Validation
- 계약
    - 업체 생성을 통해 업체 Master 정보가 생성된 업체에 한해 계약 생성 가능
    - 업체별 계약 기간이 중복되지 않도록 Validation
- 광고입찰
    - 계약생성 및 계약기간이 유효한 업체에 한해 광고입찰 가능
    - 입력된 상품ID가 광고입찰 업체가 등록한 상품ID인지 Vadliation

Spring에서는 데이터의 유효성 검증을 위해서 `@Valid` 어노테이션을 사용할 수 있는 의존성을 제공하는데 기본적인 유효성 검증 기능 Bean Validation을 제공합니다(이에 대한 내용은 [[Spring Boot] 8. 회원가입 시 유효성 검사 - 스프링 유효성 검증 기본 어노테이션 종류]({{site.url}}/application/web/spring-boot/8-join-validation/#스프링-유효성-검증-기본-어노테이션-종류) 참조). 그러나 데이터 관계에 따른 유효성 검증 기능은 제공하지 않으므로 별도로 구현해야 했습니다.

### 사용자 정의 제약(Custom Constraint)
필요한 제약을 코드로 직접 정의해서 사용할 수 있는 방법에 대해 고민하면서 관련 내용을 조사하다가 Spring Boot Validation 의존성의 ConstraintValidator 인터페이스의 존재에 대해 알게되었습니다. 해당 인터페이스는 스프링 프레임워크에서 임의의 제약(Constraint)과 검증자(Vadliator)를 구현할 수 있도록 해줍니다.

예를 들어 업체 도메인에서 업체 등록 시 해당 업체가 상품 리스트에 세팅되어 있는 업체인지 아닌지 검증하는 검증자를 다음과 같이 구현할 수 있습니다. ConstraintValidator의 제네릭(Generic) 인자에는 구현한 검증자를 사용할 어노테이션 인터페이스와 검증할 대상의 타입을 전달합니다.

```java:/src/main/.../CompanyNameExistInRepoValidator
/**
 * 업체 등록 시 상품 리스트에 셋팅된 업체명인지 아닌지 유효성 검사 진행
 * 입력된 업체명에 대해 DB에 등록되어 있는지 검사
 */
@RequiredArgsConstructor
public class CompanyNameExistInRepoValidator implements ConstraintValidator<CompanyNameExistInRepo, String> {
    private final CompanyRepository companyRepository;

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return companyRepository.existsByName(value);
    }
}
```

어노테이션 인터페이스는 `@interface`로 정의할 수 있습니다. 이 때 `@Target`, `@Retention`, `@Contraint`라는 메타 어노테이션을 사용하는데, 모두 중요한 어노테이션이지만 소스코드를 작성할 당시에 관련 내용을 자세히 숙지한 채 작성하진 못했습니다. 간단하게 해당 어노테이션이 적용될 위치를 지정하는 것이 `@Target`, 해당 어노테이션 정보에 대한 유지(retention) 정책을 지정하는 것이 `@Retention`의 역할이라고 추측했습니다. `@Constraint` 어노테이션이 유효성 검증 처리를 위해 사용하는 어노테이션으로, `validatedBy` 인자에 검증자 클래스를 넘기면 검증자 내부에서 오버라이딩된 `isValid` 메서드를 수행합니다.

```java:/src/main/.../CompanyNameExistInRepo
/**
 * 업체 등록 시 상품 리스트에 셋팅된 업체명인지 아닌지 검사하기 위한 유효성 검사 어노테이션 정의
 */
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE})
@Retention(RUNTIME)
@Constraint(validatedBy=CompanyNameExistInRepoValidator.class)
@Documented
public @interface CompanyNameExistInRepo {
    String message() default "상품 리스트에 존재하는 업체명이 아닙니다. 다시 한 번 확인해주세요.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

제약 어노테이션과 검증자를 모두 구현했으면 다음과 같이 DTO 클래스 내부에서 유효성 검증이 필요한 필드에 직접 정의한 어노테이션을 사용하여 DTO가 컨트롤러나 서비스 빈에 전달될 때 유효성 검증을 수행할 수 있습니다.

```java
/**
 * 클라이언트가 업체 도메인 관련 HTTP 요청 시 전송하는 DTO 클래스
 * Client ↔ CompanyRestController 레이어 간 전송 데이터 객체
 */
@Getter
@Setter
@ToString
@NoArgsConstructor
public class CompanyReqDto {
    @NotBlank(message="업체명을 입력해주세요.")
    @CompanyNameExistInRepo
    private String companyName;

    ...
}

/**
 * 업체 도메인 관련 HTTP request 매핑 및 처리를 위한 컨트롤러 클래스
 */
@RestController
@RequiredArgsConstructor
public class CompanyRestController {
    ...

    /**
     * 업체 등록 요청 API
     * 요청 데이터를 받아 DB에 생성, 생성된 데이터 반환
     * @param companyReqDto
     * @return ResponseEntity
     */
    @PostMapping("/api/company")
    public ResponseEntity<?> registerCompany(@RequestBody @Valid CompanyReqDto companyReqDto) {
        ...
    }

    ...
}
```

### 클래스 단위 제약(Class Level Constraint)
위에서 구현한 사용자 정의 제약은 하나의 필드에 적용되는 것이었습니다. 하지만 광고입찰 도메인에서 입력한 상품ID가 광고입찰을 요청한 업체가 등록했던 상품의 ID인지 검증하는 기능, 즉 두 개 이상의 필드를 참조한 유효성 검증을 위한 제약도 구현해야 합니다. 이는 클래스 단위 제약을 구현해서 해결할 수 있었습니다.

클래스 단위 제약은 유효성 검증을 위한 어노테이션을 필드가 아닌 클래스에 사용하면서, ConstraintValidator를 상속하여 구현한 검증자의 제네릭 인자에 클래스 타입을 명시하고 이후 오버라이딩한 `isValid` 메서드에서 클래스에서 참조가 필요한 필드에 접근해 유효성 검증을 수행하는 로직을 구현하면 됩니다.

```java:/src/main/.../CompanyOwnsProductValidator
/**
 * 광고 입찰 생성 시 입력된 업체 ID와 상품 ID에 대해 해당 업체가 해당 상품을 소유하고 있는지 검사
 * 입력받은 상품 ID로부터 상품 엔티티를 조회하고 해당 엔티티의 업체 ID가 입력받은 업체 ID와 일치하는지 검사
 */
@RequiredArgsConstructor
public class CompanyOwnsProductValidator implements ConstraintValidator<CompanyOwnsProduct, AdBidReqDto> {
    private final ProductService productService;
    private String message;

    @Override
    public void initialize(CompanyOwnsProduct constraintAnnotation) {
        message = constraintAnnotation.message();
    }

    @Override
    public boolean isValid(AdBidReqDto value, ConstraintValidatorContext context) {
        ProductDto productDto = productService.getProduct(value.getProductId());

        if (productDto.getCompanyDto().getId().equals(value.getCompanyId())) {
            return true;
        }

        context.disableDefaultConstraintViolation();
        context.buildConstraintViolationWithTemplate(message)
                .addPropertyNode("productId")
                .addConstraintViolation();

        return false;
    }
}
```

한 가지 유의할 점으로, 여러 필드를 검사한 결과를 ConstraintValidatorContext에 반영할 때 특정 필드를 선택해서 유효성 검증 결과를 반영하기 위해 `.disableDefaultConstraintViolation()`와 `.buildConstraintViolationWithTemplate()` 메서드를 이용했습니다. 클래스 단위 제약에서 유효성 검증에 실패했을 때 이와 같은 처리를 하지 않으면 특정 데이터 필드에 대한 오류가 아니라 클래스에 대한 오류를 반환받게 됩니다. 이는 클라이언트에게 유효성 검증 실패에 대한 오류 메세지를 반환받았을 때 오해할 수도 있기 때문에 위와 같이 구현했습니다.

```java:/src/main/.../CompanyOwnsProduct
/**
 * 광고 입찰 생성 시 입력된 업체 ID와 상품 ID에 대해 해당 업체가 해당 상품을 소유하고 있는지 검사하기 위한 유효성 검사 어노테이션 정의
 */
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE})
@Retention(RUNTIME)
@Constraint(validatedBy=CompanyOwnsProductValidator.class)
@Documented
public @interface CompanyOwnsProduct {
    String message() default "업체가 해당 상품을 소유하고 있지 않습니다. 업체와 상품의 소유 관계를 다시 한 번 확인해주세요.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

마찬가지로 제약 어노테이션 정의와 검증자까지 구현한 뒤 클래스에 유효성 검증 어노테이션을 사용할 수 있었습니다.

```java
/**
 * 클라이언트가 광고입찰 도메인 관련 HTTP 요청 시 전송하는 DTO 클래스
 * Client ↔ AdBidRestController 레이어 간 전송 데이터 객체
 */
@Getter
@Setter
@ToString
@NoArgsConstructor
@CompanyOwnsProduct
public class AdBidReqDto {
    @NotNull(message="업체 ID를 입력해주세요.")
    @Min(value=(long)1e9+1, message="업체 ID는 최소 10자리로 입력해주세요.")
    @InContract
    private Long companyId;

    @NotNull(message="상품 ID를 입력해주세요.")
    @Min(value=(long)1e9+1, message="상품 ID는 최소 10자리로 입력해주세요.")
    private Long productId;

    ...
}

/**
 * 광고입찰 도메인 관련 HTTP request 매핑 및 처리를 위한 컨트롤러 클래스
 */
@RestController
@RequiredArgsConstructor
public class AdBidRestController {
    ...

    /**
     * 광고입찰 생성 요청 API
     * 요청 데이터를 받아 DB에 생성, 생성된 데이터 반환
     * @param adBidReqDto
     * @return ResponseEntity
     */
    @PostMapping("/api/ad/bid")
    public ResponseEntity<?> createAdBid(@RequestBody @Valid AdBidReqDto adBidReqDto) {
        ...
    }

    ...
}
```

### 제약 그룹(Grouping)
한 가지 또 어려웠던 점은 업체(COMPANY) 도메인에서 상품 리스트에 세팅된 업체명에 대해서만 등록 가능하도록 구현하는 것이었는데, 클라이언트로부터 전달받는 업체 데이터는 동일한 DTO 객체를 사용하면서 업체 등록 요청에 대해서만 업체 사업자번호나 전화번호에 대한 유효성 검증을 수행하도록 구현하고자 했습니다. 동일한 도메인에서 상황에 따라 유효성 검증이 필요한 필드가 달라지는 경우 어떻게 구현해야 할지 이번 프로젝트를 통해 학습하고 적용해보고 싶었습니다(실제 서비스에서는 이와 같은 상황을 더 많이 마주할테니까요).

각 상황에 대해 별도로 DTO 객체를 사용하게 되면 기능 구현에 큰 문제가 없겠지만 이런 경우 중복되는 코드가 많이 발생할 것이라 예상했습니다. 조사해보니 마커 인터페이스를 활용하면 동일한 DTO 객체를 사용하면서도 상황(그룹)에 따라 원하는 필드에 대해서만 유효성 검증이 가능하다는 것을 알게됐습니다.

마커 인터페이스란 어떤 메서드도 선언하지 않은 인터페이스를 말합니다. '업체 등록' 이라는 상황을 나타내는 마커 인터페이스 `CompanyRegister`를 다음과 같이 작성해줬습니다.

```java:/src/main/.../CompanyRegister
public interface CompanyRegister {
}
```

그리고 유효성 검증을 마커 인터페이스에 해당하는 상황에 대해서 수행하겠다는 뜻으로 유효성 검증 어노테이션의 `groups` 인자에 마커 인터페이스의 클래스 정보를 넘기도록 코딩합니다.

```java:/src/main/.../CompanyReqDto
/**
 * 클라이언트가 업체 도메인 관련 HTTP 요청 시 전송하는 DTO 클래스
 * Client ↔ CompanyRestController 레이어 간 전송 데이터 객체
 */
@Getter
@Setter
@ToString
@NoArgsConstructor
public class CompanyReqDto {
    @NotBlank(message="업체명을 입력해주세요.")
    @CompanyNameExistInRepo
    private String companyName;

    @NotBlank(message="사업자번호를 입력해주세요.", groups=CompanyRegister.class)
    @BusinessRegNum(groups=CompanyRegister.class)
    @BusinessRegNumNotDuplicated(groups=CompanyRegister.class)
    private String businessRegistrationNumber;

    @NotBlank(message="업체전화번호를 입력해주세요.", groups=CompanyRegister.class)
    @PhoneNum(groups=CompanyRegister.class)
    private String phoneNumber;

    @NotBlank(message="주소지를 입력해주세요.", groups=CompanyRegister.class)
    @NullOrNotBlank(message="주소지를 입력해주세요.", groups=CompanyRegister.class)
    private String address;

    ...
}
```

이후에 해당 상황에 대한 요청을 처리하는 메서드에 `@Validated` 어노테이션을 사용하고, 어노테이션의 인자에 마커 인터페이스의 클래스 정보를 넘겨서 해당 메서드가 마커 인터페이스에 해당하는 상황을 처리하는 메서드임을 명시하면 전달받는 데이터에 대해 상황에 따른 유효성 검증을 수행하게 됩니다.

```java
/**
 * 업체 도메인 관련 HTTP request 매핑 및 처리를 위한 컨트롤러 클래스
 */
@RestController
@RequiredArgsConstructor
public class CompanyRestController {
    ...

    /**
     * 업체 등록 요청 API
     * 요청 데이터를 받아 DB에 생성, 생성된 데이터 반환
     * @param companyReqDto
     * @return ResponseEntity
     */
    @Validated(CompanyRegister.class)
    @PostMapping("/api/company")
    public ResponseEntity<?> registerCompany(@RequestBody @Valid CompanyReqDto companyReqDto) {
        ...
    }
}
```

추가로 특정 1개 상황(그룹)이 아니라 2개 이상의 상황에서 동일하게 유효성 검증을 수행하려면 어떻게 해야 할까 궁금해서 찾아보니 `groups` 인자에 다음과 같이 `{}`를 이용해서 list initialization으로 여러 개의 마커 인터페이스 클래스 정보를 넘기면 된다고 합니다. 

```java
@NotEmpty(groups={A.class, B.class}) 
```

## A. 참조
soojong, "[JPA] mappedBy 이해하기", *Tistory*, Nov. 16, 2021. [Online]. Available: [https://soojong.tistory.com/entry/JPA-mappedBy-이해하기](https://soojong.tistory.com/entry/JPA-mappedBy-이해하기) [Accessed Jun. 19, 2022].

Developer RyanKim, "(JPA) JPA 성능개선이란? 성능개선 적용기 (fetch join/BatchSize)", *Tistory*, Mar. 9, 2020. [Online]. Available: [https://lion-king.tistory.com/53](https://lion-king.tistory.com/53) [Accessed Jun. 19, 2022].

M_1, "Which SQL statement is faster? (HAVING vs. WHERE...)", *stackoverflow.com*, Nov. 30, 2008. [Online]. Available: [https://stackoverflow.com/questions/328636/which-sql-statement-is-faster-having-vs-where](https://stackoverflow.com/questions/328636/which-sql-statement-is-faster-having-vs-where) [Accessed Jun. 23, 2022].

Felix, "WHERE vs. HAVING performance with GROUP BY", *stackoverflow.com*, Apr. 10, 2018. [Online]. Available: [https://stackoverflow.com/questions/49758446/where-vs-having-performance-with-group-by](https://stackoverflow.com/questions/49758446/where-vs-having-performance-with-group-by) [Accessed Jun. 23, 2022].

JeongHoeWoon, "https://hoestory.tistory.com/33", *Tistory*, Apr. 22, 2022. [Online]. Available: [https://hoestory.tistory.com/33](https://hoestory.tistory.com/33) [Accessed Jun. 23, 2022].

nathan29849, "JUnit 5 Test가 생성자 의존성 주입을 하는 방법", *Tistory*, May. 13, 2022. [Online]. Available: [https://velog.io/@nathan29849/JUnit-Test-구조](https://velog.io/@nathan29849/JUnit-Test-구조) [Accessed Jun. 23, 2022].

backtony, "Spring REST Docs 적용 및 최적화 하기", *github.com*, Jun. 18, 2021. [Online]. Available: [https://github.com/backtony/blog-code/tree/master/spring-rest-docs](https://github.com/backtony/blog-code/tree/master/spring-rest-docs) [Accessed Jul. 26, 2022].

신진호, "Validation 어디까지 해봤니?", *NHN Cloud Meetup!*, Mar. 4, 2020. [Online]. Available: [https://meetup.toast.com/posts/223](https://meetup.toast.com/posts/223) [Accessed Jun. 19, 2022].

jongmin92, "Spring Boot에서의 Bean Validation (1)", *Github.io*, Nov. 18, 2019. [Online]. Available: [https://jongmin92.github.io/2019/11/18/Spring/bean-validation-1/](https://jongmin92.github.io/2019/11/18/Spring/bean-validation-1/) [Accessed Jun. 19, 2022].

dahye-jeong, "ITEM 41: 정의하려는 것이 타입이라면 마커 인터페이스를 사용해라", *gitbook.io*, Jun. 14, 2021. [Online]. Available: [https://dahye-jeong.gitbook.io/java/java/effective_java/2021-06-14-use-marker-interfaces-to-define-types](https://dahye-jeong.gitbook.io/java/java/effective_java/2021-06-14-use-marker-interfaces-to-define-types) [Accessed Jul. 9, 2022].

망나니개발자, "[Spring] @Valid와 @Validated를 이용한 유효성 검증의 동작 원리 및 사용법 예시 - (1/2)", *Tistory*, Jul. 19, 2021. [Online]. Available: [https://mangkyu.tistory.com/174](https://mangkyu.tistory.com/174) [Accessed Jul. 9, 2022].
