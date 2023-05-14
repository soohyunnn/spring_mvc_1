# MVC 프레임워크 만들기

## 프론트 컨트롤러 패턴 소개

**프론트 컨트롤러 도입 전**

<img width="547" alt="Untitled" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/f37208e4-ae23-4784-95b4-93a81eec3eb0">

사진과 같이 입구가 없고 아무데에서나 들어올 수 있는 구조이다.

**프론트 컨트롤러 도입 후**

<img width="544" alt="Untitled 1" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/61242057-dc3c-4567-8494-c3a5fb242a14">

프론트 컨트롤러를 도입하면 공통이 하나로 통합되어 입구가 하나가 되는 구조이다.

**FrontController 패턴 특징**

- 프론트 컨트롤러 서블릿 하나로 클라이언트의 요청을 받음
- 프론트 컨트롤러가 요청에 맞는 컨트롤러를 찾아서 호출
- 입구를 하나로 함
- 공통 처리 가능
- 프론트 컨트롤러를 제외한 나머지 컨트롤러는 서블릿을 사용하지 않아도 됨

**스프링 웹 MVC와 프론트 컨트롤러**

스프링 웹 MVC의 핵심도 바로 **FrontController**

스프링 웹 MVC의 **DispatcherServlet**이 FrontController 패턴으로 구현되어 있음

## 프론트 컨트롤러 도입 - v1

이제 프론트 컨트롤러를 단계적으로 도입해보자.

기존 코드를 최대한 유지하면서, 프론트 컨트롤러를 도입하는 것이 목표이다.

먼저 구조를 맞추어두고 점진적으로 리팩토링 해보자.

### **V1 구조**

<img width="544" alt="Untitled 2" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/c1dc89d7-bdee-4341-b376-6f7fe5c53e10">

**ControllerV1**

```java
package hello.servlet.web.frontcontroller.v1;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public interface ControllerV1 {

    void process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException;

}
```

서블릿과 비슷한 모양의 컨트롤러 인터페이스를 도입한다. 각 컨트롤러들은 이 인터페이스를 구현하면 된다. 프론트 컨트롤러는 이 인터페이스를 호출해서 구현과 관계없이 로직의 일관성을 가져갈 수 있다.

이제 이 인터페이스를 구현한 컨트롤러를 만들어보자.

**MemberFormControllerV1 - 회원 등록 컨트롤러**

```java
package hello.servlet.web.frontcontroller.v1.controller;

import hello.servlet.web.frontcontroller.v1.ControllerV1;
import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class MemberFormControllerV1 implements ControllerV1 {
    @Override
    public void process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String viewPath = "/WEB-INF/views/new-form.jsp";
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);

    }
}
```

**MemberSaveControlerV1 - 회원 저장 컨트롤러**

```java
package hello.servlet.web.frontcontroller.v1.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.v1.ControllerV1;
import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class MemberSaveControllerV1 implements ControllerV1 {

    private MemberRepository memberRepository = MemberRepository.getInstance();
    @Override
    public void process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String username = req.getParameter("username");
        int age = Integer.parseInt(req.getParameter("age"));

        Member member = new Member(username, age);
        memberRepository.save(member);

        //Model에 데이터를 보관한다.
        req.setAttribute("member", member);

        String viewPath = "/WEB-INF/views/save-result.jsp";
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);
    }
}
```

**MemberListControllerV1 - 회원 목록 컨트롤러**

```java
package hello.servlet.web.frontcontroller.v1.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.v1.ControllerV1;
import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.List;

public class MemberListControllerV1 implements ControllerV1 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public void process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        List<Member> members = memberRepository.findAll();

        req.setAttribute("members", members);

        String viewPath = "/WEB-INF/views/members.jsp";
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);
    }
}
```

내부 로직은 기존 서블릿과 동일하며 이제 프론트 컨트롤러를 만들어보자.

**FrontControllerServletV1 - 프론트 컨트롤러**

```java
package hello.servlet.web.frontcontroller.v1;

import hello.servlet.web.frontcontroller.v1.controller.MemberFormControllerV1;
import hello.servlet.web.frontcontroller.v1.controller.MemberListControllerV1;
import hello.servlet.web.frontcontroller.v1.controller.MemberSaveControllerV1;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@WebServlet(name="frontControllerServletV1", urlPatterns = "/front-controller/v1/*")
public class FrontControllerServletV1 extends HttpServlet {

    private Map<String, ControllerV1> controllerV1Map = new HashMap<>();

    public FrontControllerServletV1() {
        controllerV1Map.put("/front-controller/v1/members/new-form", new MemberFormControllerV1());
        controllerV1Map.put("/front-controller/v1/members/save", new MemberSaveControllerV1());
        controllerV1Map.put("/front-controller/v1/members", new MemberListControllerV1());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("FrontControllerServletV1.service");

        //front-controller/v1/members
        String requestURI = req.getRequestURI();

        ControllerV1 controller = controllerV1Map.get(requestURI);
        if (controller == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        controller.process(req, resp);  // controller가 있으면 process를 호출
    }
}
```

### **프론트 컨트롤러 분석**

**urlPatterns**

- `urlPatterns = “/front-controler/v1/*”` : `/front-controller/v1`를 포함한 하위 모든 요청은 이 서블릿에서 받아들인다.
- 예) `/front-controller/v1` , `/front-controller/v1/a` , `/front-control/v1/a/b`

**controllerMap**

- key : 매핑 URL
- value : 호출될 컨트롤러

**service()**

먼저 `requestURI`를 조회해서 실제 호출할 컨트롤러를 `controllerMap`에서 찾는다. 만약 없다면 404(SC_NOT_FOUND) 상태 코드를 반환한다.

컨트롤러를 찾고 `controller.process(request, response);`을 호출해서 해당 컨트롤러를 실행한다.

**JSP**

JSP는 이전 MVC에서 사용했던 것을 그대로 사용한다.

**실행**

- 등록 : [`http://localhost:8080/front-controller/v1/members/new-form`](http://localhost:8080/front-controller/v1/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v1/members`](http://localhost:8080/front-controller/v1/members)

## View 분리 - v2

아래 코드와 같이 모든 컨트롤러에서 뷰로 이동하는 부분에 중복이 있어, 이 부분을 분리할 것이다.

```java
String viewPath = "/WEB-INF/views/new-form.jsp";
RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
dispatcher.forward(request, response);
```

### V2 구조

<img width="543" alt="Untitled 3" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/0ce5505e-0e47-45cc-9f08-8d726b92c9b8">

**MyView**

뷰 객체는 이후 다른 버전에서도 함께 사용하므로 패키지 위치를 `frontcontroller`에 두었다.

```java
package hello.servlet.web.frontcontroller;

import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class MyView {
    private String viewPath;

    public MyView(String viewPath) {
        this.viewPath = viewPath;
    }

    public void render(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);
    }

}
```

**ControllerV2**

```java
package hello.servlet.web.frontcontroller.v2;

import hello.servlet.web.frontcontroller.MyView;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public interface ControllerV2 {

    MyView process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException;

}
```

**MemberFormControllerV2 - 회원 등록 폼**

```java
package hello.servlet.web.frontcontroller.v2.controller;

import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v2.ControllerV2;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class MemberFormControllerV2 implements ControllerV2 {
    @Override
    public MyView process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        return new MyView("/WEB-INF/views/new-form.jsp");
    }
}
```

이제 각 컨트롤러는 복잡한 dispatcher.forward()를 직접 생성해서 호출하지 않아도 된다. 단순히 MyView 객체를 생성하고 거기에 뷰 이름만 넣고 반환하면 된다.

ControllerV1을 구현한 클래스와 ControllerV2를 구현한 클래스를 비교해보면, 이 부분의 중복이 제거된 것을 확인할 수 있다.

**MemberSaveControllerV2 - 회원 저장**

```java
package hello.servlet.web.frontcontroller.v2.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v2.ControllerV2;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class MemberSaveControllerV2 implements ControllerV2 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public MyView process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        String username = req.getParameter("username");
        int age = Integer.parseInt(req.getParameter("age"));

        Member member = new Member(username, age);
        memberRepository.save(member);

        //Model에 데이터를 보관한다.
        req.setAttribute("member", member);

        return new MyView("/WEB-INF/views/save-result.jsp");
    }
}
```

**MemberListControlerV2 - 회원 목록**

```java
package hello.servlet.web.frontcontroller.v2.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v2.ControllerV2;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.List;

public class MemberListControllerV2 implements ControllerV2 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public MyView process(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        List<Member> members = memberRepository.findAll();
        req.setAttribute("members", members);
        return new MyView("/WEB-INF/views/members.jsp");
    }
}
```

**프론트 컨트롤러 V2**

```java
package hello.servlet.web.frontcontroller.v2;

import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v2.controller.MemberFormControllerV2;
import hello.servlet.web.frontcontroller.v2.controller.MemberListControllerV2;
import hello.servlet.web.frontcontroller.v2.controller.MemberSaveControllerV2;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@WebServlet(name="frontControllerServletV2", urlPatterns = "/front-controller/v2/*")
public class FrontControllerServletV2 extends HttpServlet {

    private Map<String, ControllerV2> controllerV2Map = new HashMap<>();

    public FrontControllerServletV2() {
        controllerV2Map.put("/front-controller/v2/members/new-form", new MemberFormControllerV2());
        controllerV2Map.put("/front-controller/v2/members/save", new MemberSaveControllerV2());
        controllerV2Map.put("/front-controller/v2/members", new MemberListControllerV2());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("FrontControllerServletV2.service");

        //front-controller/v2/members
        String requestURI = req.getRequestURI();

        ControllerV2 controller = controllerV2Map.get(requestURI);
        if (controller == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        MyView myView = controller.process(req, resp);// controller가 있으면 process를 호출
        myView.render(req, resp);
    }
}
```

ControllerV2의 반환 타입이 MyView이므로 프론트 컨트롤러는 컨트롤러의 호출 결과로 MyView를 반환 받는다. 그리고 view.render()를 호출하면 forward 로직을 수행해서 JSP가 실행된다.

`MyView.render()`

```java
public void render(HttpServletRequest request, HttpServletResponse response)
throws ServletException, IOException {
    RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
    dispatcher.forward(request, response);
}
```

프론트 컨트롤러의 도입으로 MyView객체의 render()를 호출하는 부분을 모두 일관되게 처리할 수 있다. 각각의 컨트롤러는 MyView 객체를 생성만 해서 반환하면 된다.

**실행**

- 등록 : [`http://localhost:8080/front-controller/v2/members/new-form`](http://localhost:8080/front-controller/v2/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v2/members`](http://localhost:8080/front-controller/v2/members)

## Model 추가 - v3

**서블릿 종속성 제거**

컨트로러 입장에서 HttpServletRequest, HttpServletResponse이 꼭 필요한지 생각해보자.

요청 파라미터 정보는 자바의 Map으로 대신 넘기도록 하면 지금 구조에서는 컨트롤러가 서블릿 기술을 몰라도 동작할 수 있다.

그리고 request 객체를 Model로 사용하는 대신에 별도의 Model 객체를 만들어서 반환하면 된다.

이제 컨트롤러가 서블릿 기술을 전혀 사용하지 않도록 변경해보자. 이렇게 하면 구현 코드도 단순해지고, 테스트 코드 작성도 쉬워진다.

**뷰 이름 중복 제거**

컨트롤러에서 지정하는 뷰 이름이 중복되고 있다.

컨트롤러는 **뷰의 논리 이름**을 반환하고, 실제 물리 위치의 이름은 프론트 컨트롤러에서 처리하도록 단순화 시키자.

이렇게 변경하면 향후 뷰의 폴더 위치가 함께 이동해도 프론트 컨트롤러만 고치면 된다.

- `/WEB-INF/views/new-form.jsp` → new-form
- `/WEB-INF/views/save-result.jsp` → save-result
- `/WEB-INF/views/members.jsp` → members

### V3 구조

<img width="545" alt="Untitled 4" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/984fcde3-6422-4afe-b385-8e3b6266ccc8">

**ModelView**

지금까지 컨트롤러에서 서블릿에 종속적인 HttpServletRequest를 사용했다. 그리고 Model도 `request.setAttribute()`를 통해 데이터를 저장하고 뷰에 전달했다.

서블릿의 종속성을 제거하기 위해 Model을 직접 만들고, 추가로 View 이름까지 전달하는 객체를 만들어보자.

→ 컨트롤러에서 HttpServletRequest를 사용할 수 없기 때문에 `request.setAttribute()`를 호출할 수도 없다. 따라서 Model이 별도로 필요하다.

⚠️ ModelView 객체는 다른 버전에서도 사용하므로 패키지를 `frontcontroller`에 둔다.

**ModelView**

```java
package hello.servlet.web.frontcontroller;

import lombok.Getter;
import lombok.Setter;

import java.util.HashMap;
import java.util.Map;

@Getter
@Setter
public class ModelView {

    private String viewName;
    private Map<String, Object> model = new HashMap<>();

    public ModelView(String viewName) {
        this.viewName = viewName;
    }
}
```

뷰의 이름과 뷰를 렌더링할 때 필요한 model 객체를 가지고 있다. model은 단순히 map으로 되어 있으므로 컨트롤러에서 뷰에 필요한 데이터를 key, value로 넣어주면 된다.

**ControllerV3**

```java
package hello.servlet.web.frontcontroller.v3;

import hello.servlet.web.frontcontroller.ModelView;

import java.util.Map;

public interface ControllerV3 {

    ModelView process(Map<String, String> paramMap);

}
```

이 컨트롤러는 서블릿 기술을 전혀 사용하지 않는다. 따라서 구현이 매우 단순해지고, 테스크 코드 작성시 테스트 하기 쉽다.

HttpServletRequest가 제공하는 파라미터는 프론트 컨트롤러가 paramMap에 담아서 호출해주면 된다.

응답 결과로 뷰 이름과 뷰에 전달할 Model 데이터를 포함하는 ModelView 객체를 반환하면 된다.

**MemberFormControllerV3 - 회원 등록 폼**

```java
package hello.servlet.web.frontcontroller.v3.controller;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.v3.ControllerV3;

import java.util.Map;

public class MemberFormControllerV3 implements ControllerV3 {
    @Override
    public ModelView process(Map<String, String> paramMap) {
        return new ModelView("new-form");
    }
}
```

`ModelView`를 생성할 때 `new-form`이라는 view의 논리적인 이름을 지정한다. 실제 물리적인 이름은 프론트 컨트롤러에서 처리한다.

**MemberSaveControllerV3 - 회원 저장**

```java
package hello.servlet.web.frontcontroller.v3.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.v3.ControllerV3;

import java.util.Map;

public class MemberSaveControllerV3 implements ControllerV3 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public ModelView process(Map<String, String> paramMap) {
        String username = paramMap.get("username");
        int age = Integer.parseInt(paramMap.get("age"));

        Member member = new Member(username, age);
        memberRepository.save(member);

        ModelView mv = new ModelView("save-result");
        mv.getModel().put("member", member);
        return mv;
    }
}
```

`paramMap.get(”username”);`

파라미터 정보는 map에 담겨있다. map에서 필요한 요청 파라미터를 조회하면 된다.

`mv.getModel().put(”member”, member);`

모델은 단순한 map이므로 모델에 뷰에서 필요한 `member`객체를 담고 반환한다.

**MemberListControllerV3 - 회원 목록**

```java
package hello.servlet.web.frontcontroller.v3.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.v3.ControllerV3;

import java.util.List;
import java.util.Map;

public class MemberListControllerV3 implements ControllerV3 {

    private MemberRepository memberRepository = MemberRepository.getInstance();
    @Override
    public ModelView process(Map<String, String> paramMap) {
        List<Member> members = memberRepository.findAll();
        ModelView mv = new ModelView("members");
        mv.getModel().put("members", members);

        return mv;
    }
}
```

**FrontControllerServletV3**

```java
package hello.servlet.web.frontcontroller.v3;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v2.ControllerV2;
import hello.servlet.web.frontcontroller.v2.controller.MemberFormControllerV2;
import hello.servlet.web.frontcontroller.v2.controller.MemberListControllerV2;
import hello.servlet.web.frontcontroller.v2.controller.MemberSaveControllerV2;
import hello.servlet.web.frontcontroller.v3.controller.MemberFormControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberListControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberSaveControllerV3;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@WebServlet(name="frontControllerServletV3", urlPatterns = "/front-controller/v3/*")
public class FrontControllerServletV3 extends HttpServlet {

    private Map<String, ControllerV3> controllerV3Map = new HashMap<>();

    public FrontControllerServletV3() {
        controllerV3Map.put("/front-controller/v3/members/new-form", new MemberFormControllerV3());
        controllerV3Map.put("/front-controller/v3/members/save", new MemberSaveControllerV3());
        controllerV3Map.put("/front-controller/v3/members", new MemberListControllerV3());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("FrontControllerServletV3.service");

        //front-controller/v3/members
        String requestURI = req.getRequestURI();

        ControllerV3 controller = controllerV3Map.get(requestURI);
        if (controller == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        //paramMap
        Map<String, String> paramMap = createParamMap(req);

        ModelView mv = controller.process(paramMap);// controller가 있으면 process를 호출
        String viewName = mv.getViewName();// 논리이름을 가져옴.

        MyView view = viewResolver(viewName);
        view.render(mv.getModel(), req, resp);
    }

    private static MyView viewResolver(String viewName) {
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }

    private static Map<String, String> createParamMap(HttpServletRequest req) {
        Map<String, String> paramMap = new HashMap<>();
        req.getParameterNames().asIterator()
                .forEachRemaining(paramName -> paramMap.put(paramName, req.getParameter(paramName)));
        return paramMap;
    }
}
```

`view.render(mv.getModel(), req, resp)` 코드에서 컴파일 오류가 발생할 것이다. MyView에 해당 메소드를 추가해주어야 한다.

`createParamMap()`

HttpServletRequest에 파라미터 정보를 꺼내서 Map으로 변환한다. 그리고 해당 Map(paramMap)을 컨트롤러에 전달하면서 호출한다.

**뷰 리졸버**

`MyView view = viewResolver(viewName)`

컨트롤러가 반환한 논리 뷰 이름을 실제 물리 뷰 경로로 변경한다. 그리고 실제 물리 경로가 있는 MyView 객체를 반환한다.

- 논리 뷰 이름 : `members`
- 물리 뷰 경로 : `/WEB-INF/views/member.jsp`

`view.render(mv.getModel(), req, resp)`

- 뷰 객체를 통해서 HTML 화면을 렌더링 한다.
- 뷰 객체의 `render()`는 모델 정보도 함께 받는다.
- JSP는 `request.getAttribute()`로 데이터를 조회하기 때문에, 모델의 데이터를 꺼내서 `request.setAttribute()`로 담아둔다.
- JSP로 포워드 해서 JSP를 렌더링 한다.

**MyView**

```java
package hello.servlet.web.frontcontroller;

import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.Map;

public class MyView {
    private String viewPath;

    public MyView(String viewPath) {
        this.viewPath = viewPath;
    }

    public void render(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);
    }

    public void render(Map<String, Object> model, HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        modelToRequestAttribute(model, req);
        RequestDispatcher dispatcher = req.getRequestDispatcher(viewPath);
        dispatcher.forward(req, resp);
    }

    private static void modelToRequestAttribute(Map<String, Object> model, HttpServletRequest req) {
        model.forEach((key, value) -> req.setAttribute(key, value));
    }
}
```

- 등록 : [`http://localhost:8080/front-controller/v3/members/new-form`](http://localhost:8080/front-controller/v3/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v3/members`](http://localhost:8080/front-controller/v3/members)

## 단순하고 실용적인 컨트롤러 - v4

v3 컨트롤러는 서블릿 종속성을 제거하고 뷰 경로의 중복을 제거하는 등, 잘 설계된 컨트롤러이다. 그런데 실제 컨트롤러 인터페이스를 구현하는 개발자 입장에서 보면, 항상 ModelView 객체를 생성하고 반환해야 하는 부분이 번거로울 수 있다.

좋은 프레임워크는 아키텍처도 중요하지만, 그와 더불어 실제 개발하는 개발자가 단순하고 편리하게 사용할 수 있어야 한다. → `실용성`

이제 v3를 조금 변경해서 실제 구현하는 개발자들이 매우 편리하게 개발할 수 있는 v4 버전을 개발해보자.

### V4 구조

<img width="543" alt="Untitled 5" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/cdaf5671-e37d-4dfc-be4b-d1d5558eb0af">

- 기본적인 구조는 V3와 같다. 대신 컨트롤러가 `ModelView`를 반환하지 않고, `ViewName`만 반환한다.

**ControllerV4**

```java
package hello.servlet.web.frontcontroller.v4;

import java.util.Map;

public interface ControllerV4 {

    /**
     * @param paramMap
     * @param model
     * @return viewNAme
     */
    String process(Map<String, String> paramMap, Map<String, Object> model);
}
```

ModelView 제거. model 객체는 파라미터로 전달되기 때문에 결과로 뷰의 이름만 반환해주면 된다.

이제 실제 구현 코드를 확인해보자.

**MemberFormControllerV4**

```java
package hello.servlet.web.frontcontroller.v4.controller;

import hello.servlet.web.frontcontroller.v4.ControllerV4;

import java.util.Map;

public class MemberFormControllerV4 implements ControllerV4 {

    @Override
    public String process(Map<String, String> paramMap, Map<String, Object> model) {
        return "new-form";
    }
}
```

심플하게 `new-form` 이라는 뷰의 논리 이름만 반환하면 된다.

**MemberSaveControllerV4**

```java
package hello.servlet.web.frontcontroller.v4.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.v4.ControllerV4;

import java.util.Map;

public class MemberSaveControllerV4 implements ControllerV4 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public String process(Map<String, String> paramMap, Map<String, Object> model) {
        String username = paramMap.get("username");
        int age = Integer.parseInt(paramMap.get("age"));

        Member member = new Member(username, age);
        memberRepository.save(member);

        model.put("member", member);
        return "save-result";
    }

}
```

`model.put(”member”, member)`

모델이 파라미터로 전달되기 때문에, 모델을 직접 생성하지 않아도 된다.

**MemberListControllerV4**

```java
package hello.servlet.web.frontcontroller.v4.controller;

import hello.servlet.domain.member.Member;
import hello.servlet.domain.member.MemberRepository;
import hello.servlet.web.frontcontroller.v4.ControllerV4;

import java.util.List;
import java.util.Map;

public class MemberListControllerV4 implements ControllerV4 {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public String process(Map<String, String> paramMap, Map<String, Object> model) {
        List<Member> members = memberRepository.findAll();

        model.put("members", members);
        return "members";
    }

}
```

**FrontControllerServletV4**

```java
package hello.servlet.web.frontcontroller.v4;

import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v4.controller.MemberFormControllerV4;
import hello.servlet.web.frontcontroller.v4.controller.MemberListControllerV4;
import hello.servlet.web.frontcontroller.v4.controller.MemberSaveControllerV4;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@WebServlet(name="frontControllerServletV4", urlPatterns = "/front-controller/v4/*")
public class FrontControllerServletV4 extends HttpServlet {

    private Map<String, ControllerV4> controllerV4Map = new HashMap<>();

    public FrontControllerServletV4() {
        controllerV4Map.put("/front-controller/v4/members/new-form", new MemberFormControllerV4());
        controllerV4Map.put("/front-controller/v4/members/save", new MemberSaveControllerV4());
        controllerV4Map.put("/front-controller/v4/members", new MemberListControllerV4());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("FrontControllerServletV4.service");

        //front-controller/v3/members
        String requestURI = req.getRequestURI();

        ControllerV4 controller = controllerV4Map.get(requestURI);
        if (controller == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        //paramMap
        Map<String, String> paramMap = createParamMap(req);
        Map<String, Object> model = new HashMap<>();

        String viewName = controller.process(paramMap, model);// controller가 있으면 process를 호출

        MyView view = viewResolver(viewName);
        view.render(model, req, resp);
    }

    private static MyView viewResolver(String viewName) {
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }

    private static Map<String, String> createParamMap(HttpServletRequest req) {
        Map<String, String> paramMap = new HashMap<>();
        req.getParameterNames().asIterator()
                .forEachRemaining(paramName -> paramMap.put(paramName, req.getParameter(paramName)));
        return paramMap;
    }
}
```

`frontControllerServletV4`는 이전 버전과 거의 동일하다.

**모델 객체 전달**

`Map<String, Object> model = new HashMap<>();  // 추가`

모델 객체를 프론트 컨트롤러에서 생성해서 넘겨준다. 컨트롤러에서 모델 객체에 값을 담으면 여기에 그대로 담겨있게 된다.

**뷰의 논리 이름을 직접 반환**

```java
String viewName = controller.process(paramMap, model);
MyView view = viewResolver(viewName);
```

컨트롤러가 직접 뷰의 논리 이름을 반환하므로 이 값을 사용해서 실제 물리 뷰를 찾을 수 있다.

- 등록 : [`http://localhost:8080/front-controller/v4/members/new-form`](http://localhost:8080/front-controller/v4/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v4/members`](http://localhost:8080/front-controller/v4/members)

> 정리
이번 버전의 컨트롤러는 매우 단순하고 실용적이다. 기존 구조에서 모델을 파라미터로 넘기고, 뷰의 논리 이름을 반환한다는 작은 아이디어를 적용했을 뿐인데, 컨트롤러를 구현하는 개발자 입장에서 보면 깔끔한 코드를 작성할 수 있다.
> 

결론적으로 **프레임워크나 공통 기능이 수고로워야 사용하는 개발자가 편리해진다.**

## 유연한 컨트롤러1 - v5

만약 어떤 개발자는 `ControllerV3` 방식으로 개발하고 싶고, 어떤 개발자는 `ControllerV4` 방식으로 개발하고 싶으면 어떻게 해야할까?

```java
public interface ControllerV3 {
    ModelView process(Map<String, String> paramMap);
}
```

```java
public interface ControllerV4 {
    String process(Map<String, String> paramMap, Map<String, Object> model);
}
```

**어댑터 패턴**

지금까지 개발한 프론트 컨트롤러는 한가지 방식의 컨트롤러 인터페이스만 사용할 수 있다.

ControllerV3, ControllerV4는 완전히 다른 인터페이스이다. 따라서 호환이 불가능하다. 이럴 때 사용하는 것이 바로 어댑터이다.

어댑터 패턴을 사용해서 프론트 컨트롤러가 다양한 방식의 컨트롤러를 처리할 수 있도록 변경해보자.

### V5 구조

<img width="542" alt="Untitled 6" src="https://github.com/soohyunnn/spring_mvc_1/assets/58289675/3315f1e8-ed5f-498e-97f9-049797ef9ee6">

- **핸들러 어댑터** : 중간에 어댑터 역할을 하는 어댑터가 추가되었는데 이름이 핸들러 어댑터이다. 여기서 어댑터 역할을 해주는 덕분에 다양한 종류의 컨트롤러를 호출할 수 있다.
- **핸들러** : 컨트롤러의 이름을 더 넓은 범위인 핸들러로 변경했다. 그 이유는 이제 어댑터가 있기 때문에 꼭 컨트롤러의 개념 뿐만 아니라 어떠한 것이든 해당하는 종류의 어댑터만 있으면 다 처리할 수 있기 때문이다.

**MyHandlerAdapter**

어댑터는 이렇게 구현해야 한다는 어댑터용 인터페이스이다.

```java
package hello.servlet.web.frontcontroller.v5;

import hello.servlet.web.frontcontroller.ModelView;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public interface MyHandlerAdapter {

    boolean supports(Object handler);

    ModelView handle(HttpServletRequest req, HttpServletResponse resp, Object handler) throws ServletException, IOException;
}
```

`booleans supports(Object handler)`

- handler는 컨트롤러를 말한다.
- 어댑터가 해당 컨트롤러를 처리할 수 있는지 판단하는 메서드다.

`ModelView handle(HttpServletRequest req, HttpServletResponse resp, Object handler) throws ServletException, IOException`

- 어댑터는 실제 컨트롤러를 호출하고, 그 결과로 ModelView를 반환해야 한다.
- 실제 컨트롤러가 ModelView를 반환하지 못하면, 어댑터가 ModelView를 직접 생성해서라도 반환해야 한다.
- 이전에는 프론트 컨트롤러가 실제 컨트롤러를 호출했지만 이제는 이 어댑터를 통해서 실제 컨트롤러가 호출된다.

이제 실제 어댑터를 구현해보자.

우선 ControllerV3를 지원하는 어댑터를 구현하자.

**ControllerV3HandlerAdapter**

```java
package hello.servlet.web.frontcontroller.v5.adapter;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.v3.ControllerV3;
import hello.servlet.web.frontcontroller.v5.MyHandlerAdapter;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class ControllerV3HandlerAdapter implements MyHandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return (handler instanceof ControllerV3);
    }

    @Override
    public ModelView handle(HttpServletRequest req, HttpServletResponse resp, Object handler) throws ServletException, IOException {
        ControllerV3 controller = (ControllerV3) handler;

        Map<String, String> paramMap = createParamMap(req);
        ModelView mv = controller.process(paramMap);

        return mv;
    }

    private static Map<String, String> createParamMap(HttpServletRequest req) {
        Map<String, String> paramMap = new HashMap<>();
        req.getParameterNames().asIterator()
                .forEachRemaining(paramName -> paramMap.put(paramName, req.getParameter(paramName)));
        return paramMap;
    }

}
```

👇

```java
@Override
public boolean supports(Object handler) {
    return (handler instanceof ControllerV3);
}
```

`ControllerV3`을 처리할 수 있는 어댑터를 뜻한다.

```java
ControllerV3 controller = (ControllerV3) handler;

Map<String, String> paramMap = createParamMap(req);
ModelView mv = controller.process(paramMap);

return mv;
```

handler를 컨트롤러 V3로 변환한 다음에 V3 형식에 맞도록 호출한다.

supports()를 통해 ControllerV3만 지원하기 때문에 타입 변환은 걱정없이 실행해도 된다.

ControllerV3는 ModelView를 반환하므로 그대로 ModelView를 반환하면 된다.

**FrontControllerServletV5**

```java
package hello.servlet.web.frontcontroller.v5;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v3.ControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberFormControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberListControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberSaveControllerV3;
import hello.servlet.web.frontcontroller.v5.adapter.ControllerV3HandlerAdapter;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@WebServlet(name = "frontControllerServletV5", urlPatterns = "/front-controller/v5/*")
public class FrontControllerServletV5 extends HttpServlet {

    private final Map<String, Object> handlerMappingMap = new HashMap<>();
    private final List<MyHandlerAdapter> handlerAdapters = new ArrayList<>();

    public FrontControllerServletV5() {
        initHandlerMappingMap();
        initHandlerAdapters();
    }

    private void initHandlerAdapters() {
        handlerAdapters.add(new ControllerV3HandlerAdapter());
    }

    private void initHandlerMappingMap() {
        handlerMappingMap.put("/front-controller/v5/v3/members/new-form", new MemberFormControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members/save", new MemberSaveControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members", new MemberListControllerV3());
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Object handler = getHandler(req);

        if (handler == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        MyHandlerAdapter adapter = getHandlerAdapter(handler);

        ModelView mv = adapter.handle(req, resp, handler);
        String viewName = mv.getViewName();

        MyView view = viewResolver(viewName);
        view.render(mv.getModel(), req, resp);
    }

    private static MyView viewResolver(String viewName) {
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }

    // handler 찾기
    private Object getHandler(HttpServletRequest req) {
        String requestURI = req.getRequestURI();
        return handlerMappingMap.get(requestURI);
    }
    private MyHandlerAdapter getHandlerAdapter(Object handler) {
        for (MyHandlerAdapter adapter : handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
        throw new IllegalArgumentException("handler adapter를 찾을 수 없습니다." + handler);
    }
}
```

**컨트롤러(Controller)  → 핸들러(Handler)**

이전에는 컨트롤러를 직접 매핑해서 사용했다. 그런데 이제는 어댑터를 사용하기 때문에, 컨트롤러 뿐만 아니라 어댑터가 지원하기만 하면, 어떤 것이라도 URL에 매핑해서 사용할 수 있다. 그래서 이름을 컨트롤러에서 더 넓은 범위의 핸들러로 변경했다.

**생성자**

```java
public FrontControllerServletV5() {
    initHandlerMappingMap();  // 핸들러 매핑 초기화
    initHandlerAdapters();  // 어댑터 초기화
}
```

생성자는 핸들러 매핑과 어댑터를 초기화(등록)한다.

**매핑 정보**

`private final Map<String, Object> handlerMappingMap = new HashMap<>();`

매핑 정보의 값이 `ControllerV3`, `ControllerV4` 같은 인터페이스에서 아무 값이나 받을 수 있는 Object로 변경되었다.

**핸들러 매핑**

`Object handler = getHandler(req)`

```java
private Object getHandler(HttpServletRequest req) {
    String requestURI = req.getRequestURI();
    return handlerMappingMap.get(requestURI);
}
```

핸들러 매핑 정보인 `handlerMappingMap`에서 URL에 매핑된 핸들러(컨트롤러) 객체를 찾아서 반환한다.

**핸들러를 처리할 수 있는 어댑터 조회**

`MyHandlerAdapter adapter = getHandlerAdapter(handler)`

```java
for (MyHandlerAdapter adapter : handlerAdapters) {
    if (adapter.supports(handler)) {
        return adapter;
    }
}
```

`handler`를 처리할 수 있는 어댑터를 `adapter.supports(handler)`를 통해서 찾는다.

handler가 `ControllerV3` 인터페이스를 구현했다면, `ControllerV3HandlerAdapter` 객체가 반환된다.

**어댑터 호출**

`ModelView mv = adapter.handle(req, resp, handler)`

어댑터의 `handle(req, resp, hander)` 메서드를 통해 실제 어댑터가 호출된다.

어댑터는 handler(컨트롤러)를 호출하고 그 결과를 어댑터에 맞추어 반환한다.

`ControllerV3HandlerAdapter`의 경우 어댑터의 모양과 컨트롤러의 모양이 유사해서 변환 로직이 단순하다.

- 등록 : [`http://localhost:8080/front-controller/v5/v3/members/new-form`](http://localhost:8080/front-controller/v5/v3/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v5/v3/members`](http://localhost:8080/front-controller/v5/v3/members)

## 유연한 컨트롤러2 - v5

FrontControllerServletV5에 ControllerV4 기능도 추가해보자.

**FrontControllerServletV5**

```java
package hello.servlet.web.frontcontroller.v5;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.MyView;
import hello.servlet.web.frontcontroller.v3.controller.MemberFormControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberListControllerV3;
import hello.servlet.web.frontcontroller.v3.controller.MemberSaveControllerV3;
import hello.servlet.web.frontcontroller.v4.controller.MemberFormControllerV4;
import hello.servlet.web.frontcontroller.v4.controller.MemberListControllerV4;
import hello.servlet.web.frontcontroller.v4.controller.MemberSaveControllerV4;
import hello.servlet.web.frontcontroller.v5.adapter.ControllerV3HandlerAdapter;
import hello.servlet.web.frontcontroller.v5.adapter.ControllerV4HandlerAdapter;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@WebServlet(name = "frontControllerServletV5", urlPatterns = "/front-controller/v5/*")
public class FrontControllerServletV5 extends HttpServlet {

    private final Map<String, Object> handlerMappingMap = new HashMap<>();
    private final List<MyHandlerAdapter> handlerAdapters = new ArrayList<>();

    public FrontControllerServletV5() {
        initHandlerMappingMap();
        initHandlerAdapters();
    }

    private void initHandlerMappingMap() {
        handlerMappingMap.put("/front-controller/v5/v3/members/new-form", new MemberFormControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members/save", new MemberSaveControllerV3());
        handlerMappingMap.put("/front-controller/v5/v3/members", new MemberListControllerV3());

				// V4 추가
        handlerMappingMap.put("/front-controller/v5/v4/members/new-form", new MemberFormControllerV4());
        handlerMappingMap.put("/front-controller/v5/v4/members/save", new MemberSaveControllerV4());
        handlerMappingMap.put("/front-controller/v5/v4/members", new MemberListControllerV4());
    }

    private void initHandlerAdapters() {
        handlerAdapters.add(new ControllerV3HandlerAdapter());
        handlerAdapters.add(new ControllerV4HandlerAdapter());  // V4 추가
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Object handler = getHandler(req);

        if (handler == null) {
            resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        MyHandlerAdapter adapter = getHandlerAdapter(handler);

        ModelView mv = adapter.handle(req, resp, handler);
        String viewName = mv.getViewName();

        MyView view = viewResolver(viewName);
        view.render(mv.getModel(), req, resp);
    }

    private static MyView viewResolver(String viewName) {
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");
    }

    // handler 찾기
    private Object getHandler(HttpServletRequest req) {
        String requestURI = req.getRequestURI();
        return handlerMappingMap.get(requestURI);
    }
    private MyHandlerAdapter getHandlerAdapter(Object handler) {
        for (MyHandlerAdapter adapter : handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
        throw new IllegalArgumentException("handler adapter를 찾을 수 없습니다." + handler);
    }
}
```

핸들러 매핑(`handlerMappingMap`)에 `ControllerV4`를 사용하는 컨트롤러를 추가하고, 해당 컨트롤러를 처리할 수 있는 어댑터인 `ControllerV4HandlerAdapter`도 추가한다.

**ControllerV4HandlerAdapter**

```java
package hello.servlet.web.frontcontroller.v5.adapter;

import hello.servlet.web.frontcontroller.ModelView;
import hello.servlet.web.frontcontroller.v4.ControllerV4;
import hello.servlet.web.frontcontroller.v5.MyHandlerAdapter;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class ControllerV4HandlerAdapter implements MyHandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return (handler instanceof ControllerV4);
    }

    @Override
    public ModelView handle(HttpServletRequest req, HttpServletResponse resp, Object handler) throws ServletException, IOException {
        ControllerV4 controller = (ControllerV4) handler;

        Map<String, String> paramMap = createParamMap(req);
        HashMap<String, Object> model = new HashMap<>();

        String viewName = controller.process(paramMap, model);

        ModelView mv = new ModelView(viewName);  // view 셋팅
        mv.setModel(model); // model 셋팅

        return mv;
    }

    private static Map<String, String> createParamMap(HttpServletRequest req) {
        Map<String, String> paramMap = new HashMap<>();
        req.getParameterNames().asIterator()
                .forEachRemaining(paramName -> paramMap.put(paramName, req.getParameter(paramName)));
        return paramMap;
    }
}
```

👇 

```java
public boolean supports(Object handler) {
    return (handler instanceof ControllerV4);
}
```

`handler`가 `ControllerV4`인 경우에만 처리하는 어댑터이다.

**실행 로직**

```java
ControllerV4 controller = (ControllerV4) handler;

Map<String, String> paramMap = createParamMap(req);
HashMap<String, Object> model = new HashMap<>();

String viewName = controller.process(paramMap, model);
```

handler를 ControllerV4로 캐스팅하고, paramMap, model을 만들어서 해당 컨트롤러를 호출한다. 그리고 viewName을 반환 받는다.

**어댑터 변환**

```java
ModelView mv = new ModelView(viewName);  // view 셋팅
mv.setModel(model); // model 셋팅

return mv;
```

어댑터가 호출하는 `ControllerV4`는 뷰의 이름을 반환한. 그런데 어댑터는 뷰의 이름이 아니라 `ModelView`를 만들어서 반환해야 한다. 그래서 어댑터가 꼭 필요하다.

`ControllerV4`는 뷰의 이름을 반환했지만, 어댑터는 이것을 ModelView로 만들어서 형식을 맞추어 반환한다.

**어댑터와 ControllerV4**

```java
package hello.servlet.web.frontcontroller.v4;

import java.util.Map;

public interface ControllerV4 {

    /**
     * @param paramMap
     * @param model
     * @return viewNAme
     */
    String process(Map<String, String> paramMap, Map<String, Object> model);
}
```

```java
package hello.servlet.web.frontcontroller.v5;

import hello.servlet.web.frontcontroller.ModelView;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public interface MyHandlerAdapter {

    ModelView handle(HttpServletRequest req, HttpServletResponse resp, Object handler) throws ServletException, IOException;
}
```

- 등록 : [`http://localhost:8080/front-controller/v5/v4/members/new-form`](http://localhost:8080/front-controller/v5/v4/members/new-form)
- 목록 : [`http://localhost:8080/front-controller/v5/v4/members`](http://localhost:8080/front-controller/v5/v4/members)

## 정리

**v1 : 프론트 컨트롤러를 도입**

- 기존 구조를 최대한 유지하면서 프론트 컨트롤러를 도입

**v2 : View 분류**

- 단순 반복 되는 뷰 로직 분리

**v3 : Model 추가**

- 서블릿 종속성 제거
- 뷰 이름 중복 제거

**v4 : 단순하고 실용적인 컨트롤러**

- v3와 거의 비슷
- 구현 입장에서 ModelView를 직접 생성해서 반환하지 않도록 편리한 인터페이스 제공

**v5 : 유연한 컨트롤러**

- 어댑터 도입
- 어댑터를 추가해서 프레임워크를 유연하고 확장성 있게 설계

여기에 어노테이션을 사용해서 컨트롤러를 더 편리하게 발전시킬 수도 있다. 만약 어노테이션을 사용해서 컨트롤러를 편리하게 사용할 수 있게 하려면 어떻게 해야할까? 👉 바로 어노테이션을 지원하는 어댑터를 추가하면 된다!

다형성과 어댑터 덕분에 기존 구조를 유지하면서, 프레임워크의 기능을 확장할 수 있다.

❣️ 스프링 MVC는 지금까지 학습한 내용과 거의 같은 구조를 가지고 있다.