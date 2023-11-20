# 섹션 1. API 개발 기본

## 회원 등록 API

강의에서는 템플릿 엔진을 사용하는 컨트롤러와 API 스타일의 컨트롤러의 패키지를 분리한다.   
분리하는 이유는 공통으로 예외 처리를 하는 경우 패키지 단위로 처리하는 경우가 많은데, API와 화면은 공통처리하는 요소가 매우 다르기 때문이다.
예를 들어, 화면에서는 문제 발생 시 에러 화면(html)이 나와야 하지만, API에서는 JSON spec이 나와야 한다.  

- @Controller와 @ReponseBody 어노테이션을 합친 것이 @RestController 어노테이션이다.


- @ResponseBody 는 데이터 자체를 XML이나 JSON으로 보낼 때 주로 사용하는 어노테이션이다. 


- @RequestBody 는 XML, JSON으로 온 body를 매핑해 넣어주는 역할을 한다. 

@RequestBody를 이용해 사용자로부터 name을 입력받아 회원가입한 뒤 id를 반환하는 saveMemberV1 메서드를 생성한다.  
그러나 이 방식은 치명적인 단점이 있다. 우선 화면에 노출되는 프레젠테이션 계층의 검증을 위한 것이 엔티티에 들어가는 것이므로 바람직하지 않고, 엔티티의 name을 username으로 변경하는 경우는 api 스펙 자체가 바뀌게 된다는 문제가 있다.   
또한, 실무에서는 여러 개의 등록 API가 만들어질 확률이 높다. (간편 가입, 소셜 가입 등) 엔티티 하나만으로 감당하기 어렵고 엔티티를 외부에 노출하는 것은 권장하지 않는다.  
결론적으로, API 스펙을 위한 별도의 DTO를 만들어 넘겨 주는 것이 추후 발생할 장애를 대비할 수 있는 방법이다!

v2의 경우처럼 CreateMemberRequest로 member 정보를 받아오게 되면 엔티티가 바뀌더라도 컴파일 오류를 통해 api의 영향을 받지 않고 안정적으로 운영할 수 있다.   
또한, API 스펙에 맞는 DTO를 fit하게 만들 수 있어 유지보수도 용이하다는 장점이 있다. 

## 회원 수정 API

회원 수정의 경우 같은 수정을 여러번 해도 결과가 같다(멱등하다). 
uri에 PathVariable로 수정할 id값을 받아오고, 등록과 마찬가지로 별도의 Request, Response와 DTO 생성해 반환한다.

회원 수정의 경우 트랜잭션이 일어나는 서비스 내에 구현해주고, 커맨드와 쿼리를 분리하기 위해 void 타입으로 구현한다.

## 회원 조회 API

단순 조회이고, 매번 데이터를 다시 넣기 귀찮으므로 application.yml 파일의 ddl-autofmf none으로 변경해준다. 데이터베이스 내의 데이터를 계속 사용할 수 있다. 

```java
@GetMapping("/api/v1/members")
    public List<Member> membersv1() {
        return memberService.findMembers();
    }
```

로 회원을 조회할 수 있으나 이는 몇 가지 문제점이 있다. 앞의 등록 예제에서 강조했듯, 엔티티의 정보를 외부에 노출하는 것은 매우 위험하다.   
@JsonIgnore와 같은 방식으로 정보를 숨길 수 있으나 이는 다른 API를 생성할 때 문제가 된다. 
예를 들어 회원을 조회할 때 주문 정보가 노출되지 않기 위해 orders에 어노테이션을 추가했는데 주문 정보가 필요한 관련 API를 생성해야 하는 경우가 발생할 수 있다는 것이다. 

다음과 같이 조회 기능을 작성하는 것이 바람직하다!

```java
@GetMapping("/api/v2/members")
    public Result memberV2() {
        List<Member> findMembers = memberService.findMembers();
        List<MemberDto> collect = findMembers.stream()
                .map(m -> new MemberDto(m.getName()))
                .toList();

        return new Result(collect);
    }

    @Data
    @AllArgsConstructor
    static class Result<T> {
        private T data;
    }
    
    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String name;
    }
```

특히, 여기서 MemberDTO를 바로 JSON 반환 타입으로 내보내는 것이 아닌 Result로 껍데기를 한번 씌워주는 것에 주목하자. Count를 추가한다든지 새로운 상황이 주어졌을 때 유연성있게 대처하기 위해서이다. 
또한 가급적이면 **필요한 것만 노출** 하자!