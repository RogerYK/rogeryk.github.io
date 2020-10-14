---
layout: post
title: spring boot 参数校验
category: spring boot
tags: 参数校验
description: 参数校验是web开发中必不可少的一步，无论前端后端，参数校验都是非常有必要的。不过如果每一个参数都自己手动检查就太麻烦了，实际上，我们有许多现成的工具可以使用。比如在Spring boot 中利用Java的Validation规范来进行参数检验。
---

## 问题背景

参数验证在web开发中算是必不可少的一步，通常情况下前端和后端都需要对参数进行验证。前端验证可以快速给用户反馈,给用户更好的用户体验。然而后端的参数验证也是必不可少的，因为我们的接口可能被别人发现，然后被发送请求，如果不进行验证，就有可能导致系统错误，或将错误的数据保存。前端的参数验证我们一般都会使用各种插件来帮助我们完成，比如jQuery Validation插件，以及现在使用React, Vue或Angular的Form组件一般都集成了参数验证的功能。那么后端呢。实际上Java 也有类似的解决方法。JSR 303 和 JSR 380 就是Java 对 Java Bean属性验证的一个规范。

## 手动参数验证

一般新手在进行参数验证的时候，常常会写出如下的代码。
![](/images/874963-20180425161020290-269762569.png)
虽然没什么错，不过我想应该没有多少人想天天写这样的代码。

## 使用Java 中Validation规范进行参数验证

前面我们已经提到了，JSR 303 和 JSR 380中规定了Java 中的Validation规范，其中包含了一些常用的参数验证规则.下面的表格来自validation-api-2.0.0.Final.jar  
  
| Constraint  | 详细信息 |  
| ----------- | ------- |  
| @AssertFalse | 该值必须为False |   
| @AssertTrue | 该值必须为True |  
| @DecimalMax(value，inclusive) | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 ，inclusive
表示是否包含该值 |  
| @DecimalMin(value，inclusive) | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 ，inclusive
表示是否包含该值 |  
| @Digits | 限制必须为一个小数，且整数部分的位数不能超过integer，小数部分的位数不能超过fraction |  
| @Email | 该值必须为邮箱格式 |  
| @Future | 被注释的元素必须是一个将来的日期 |  
| @FutureOrPresent | 被注释的元素必须是一个现在或将来的日期 |  
| @Max(value) | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |  
| @Min(value) | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |  
| @Negative | 该值必须小于0 |  
| @NegativeOrZero | 该值必须小于等于0 |  
| @NotBlank | 该值不为空字符串，例如"  " |  
| @NotEmpty | 该值不为空字符串 |  
| @NotNull | 该值不为Null |  
| @Null | 该值必须为Null |  
| @Past | 被注释的元素必须是一个过去的日期 |  
| @PastOrPresent | 被注释的元素必须是一个过去或现在的日期 |  
| @Pattern(regexp) | 匹配正则 |  
| @Positive | 该值必须大于0 |  
| @PositiveOrZero | 该值必须大于等于0 |  
| @Size(min,max) | 数组大小必须在[min,max]这个区间 |  

其中Spring Validator和Hibernate Validator都实现了上述的规范，默认spring boot 2中已经集成了Hibernate validator。所以我们直接使用即可。例如我们有一个用户注册的接口。前端将用户的手机号和密码发送过来，我们对参数进行检验。  
首先创建一个Form 类，如下
![](/images/210209.png)
可以看到我们对手机号和密码都设定了验证规则，然后我们的注册接口方法如下。验证的结果会传到bindResult对象中
![](/images/210233.png)
之后调用接口测试，
![](/images/210304.png)

大致使用就是这样，不过如果每个方法都这样的话，会存在很多重复的处理bindResult对象的方法。我们可以去掉这个对象，然后再验证出错的情况下, 默认会抛出一个ConstraintViolationException我们可以统一捕捉这个异常，然后返回错误参数信息。另外要注意，如果我们使用@RequestBody修饰参数时，默认检验失败时，会抛出一个MethodArgumentNotValidException异常，这是因为Spring mvc对于@ResquestBody参数，spring mvc 使用MappingJackson2HttpMessageConverter进行转换，它会在内部捕捉ContraintViolationException然后抛出MethodArgumentNotValidException异常，同理我们在使用表单提交时，会抛出BindException异常。我们也要将这些异常一起捕获。

如下：
![](/images/212840.png)

## @Validated 和 @Valid

此外， Spring 还给我们提供了一个@Validated注解，这个注解并不是JSR规范中，是Spring 自己支持的，这个注解可以放在Controller类上，这样我们就可以对方法的参数也进行约束了，这样，我们就可以将刚才的方法改写如下
![](/images/215051.png)
使用上更为灵活了，其他的差别可以查看 <https://blog.csdn.net/wangjiangongchn/article/details/86477386>

## 自定义参数验证

尽管默认已经定义了许多参数验证规则，不过还是有一些情况下需要我们自定义参数验证规则。比如位置的验证，如下, 我们要对用户传入的位置(provinceId, cityId, areaId)进行验证，必须是我们数据库中已经储存的合法的位置Id。

```java
@Data
public class AddressForm {

    private Integer id;

    @NotEmpty
    private String name;

    @Position
    private Integer provinceId;

    @Position
    private Integer cityId;

    @Position
    private Integer areaId;

    @NotEmpty
    private String address;

    @Pattern(regexp = "^\\d{11}$")
    private String mobile;

    @NotNull
    private Boolean isDefault;

}

```

让我们看一下我们自定义注解@Position的定义

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PositionValidator.class)
public @interface Position {

    String message() default  "位置错误";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

```

记得注解中的这几个方法必须要有，否则会出错。其中我们使用@Constraint注解指定了我们自定义的验证器。

接下来我们看一下我们验证器的定义

```java
public class PositionValidator implements ConstraintValidator<Position, Integer> {

    @Autowired
    private LitemallRegionService regionService;

    @Override
    public boolean isValid(Integer provinceId, ConstraintValidatorContext constraintValidatorContext) {
        if (provinceId == null) return false;
        return regionService.findById(provinceId) != null;
    }
}

```

ConstraintValidator<A, T>的定义如下

```java
public interface ConstraintValidator<A extends Annotation, T> {
    default void initialize(A constraintAnnotation) {
    }

    boolean isValid(T var1, ConstraintValidatorContext var2);
}
```

其中initialize方法传入注解，可以在此获得注解的属性值。isValid方法传入属性值进行验证。当然，上述代码直接运行会报错，regionService没有被正确的注入，值为null。这是为什么呢？因为我们使用的validator默认使用的是hibernate validator其ConstraintValidatorFactory默认在生成ConstraintValidator对象时是不会进行依赖注入的。我们需要配置一下，使用SpringConstraintValidatorFactory来生成ConstraintValidatorFactory。

如下

```java
    @Bean
    public Validator validator(final AutowireCapableBeanFactory autowireCapableBeanFactory) {
        ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
                .configure().constraintValidatorFactory(new SpringConstraintValidatorFactory(autowireCapableBeanFactory))
                .buildValidatorFactory();
        Validator validator = validatorFactory.getValidator();

        return validator;
    }

```

这样我们自定义的Validator其中的字段就会被正确注入依赖值了。

