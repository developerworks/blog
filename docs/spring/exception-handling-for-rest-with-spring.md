# Spring REST API 的异常处理

Spring 中处理异常有多种方式, 每一种方式适用于不同的情况. 3.2之前, 我们只有 HandlerExceptionResolver 和 @ExceptionHandler 可以用来处理异常. 器结果是, 整个应用程序中
导出分布着 @ExceptionHandler 注解, 异常处理分散在各个组件当中. 3.2之后, 为了解决整个问题, 引入了一个新的 @ControllerAdvice, 可以集中处理应用程序中所有异常.

## 通过控制器级别的 @ExceptionHandler

## HandlerExceptionResolver

## @ControllerAdvice(Spring 3.2及其以上版本)

## ResponseStatusException (Spring 5及其以上版本)

## Spring Security 中的访问拒绝异常