## 定义
封装一些作用于某种数据结构中的各元素的操作，它可以在不改变这个数据结构的前提下定义作用于这些元素的操作。

## 使用场景
（1）对象结构比较稳定，但经常要在此对象结构上定义新的操作。

（2）需要对一个对象结构中的对象进行很多不同的并且不相关的操作，而需要避免这些操作“污染”这些对象的类，也不希望在增加新操作时修改这些类。


## Android源码中的访问者模式

##### 编译期注解工作原理：APT（Annotation Processing Tools）

p311

注明的ButterKnife,Dagger,Retrofit等开源库都是基于APT。
编译时注解解析的基本原理是，在某些代码元素上（如类型，函数，字段等）添加注解，在编译时编译器会检查AbstractProcessor的子类，并且调用该类型的process函数，然后将添加了注解的所有元素都传递到process函数中，使得开发人员可以在编译期进行相应的处理，例如，根据注解生成新的Java类，这也是ButterKnife等开源框架的基本原理。

在编译处理的时候，是分开进行的。如果在某个处理中产生了新的Java源文件，那么就需要另外一个处理来处理新生成的源文件，如此往复，直到没有新文件被生成为止。

```
public interface Element extends javax.lang.model.AnnotatedConstruct {

    TypeMirror asType();

    ElementKind getKind();

    Set<Modifier> getModifiers();

    Name getSimpleName();

    Element getEnclosingElement();

    List<? extends Element> getEnclosedElements();

    @Override
    boolean equals(Object obj);

    @Override
    int hashCode();

    @Override
    List<? extends AnnotationMirror> getAnnotationMirrors();

    @Override
    <A extends Annotation> A getAnnotation(Class<A> annotationType);

    /**
     * Applies a visitor to this element.
     *
     * @param <R> the return type of the visitor's methods
     * @param <P> the type of the additional parameter to the visitor's methods
     * @param v   the visitor operating on this element
     * @param p   additional parameter to the visitor
     * @return a visitor-specified result
     */
    <R, P> R accept(ElementVisitor<R, P> v, P p);
}
```
accept函数接收一个ElementVisitor和类型为P的参数，ElementVisitor就是访问者类型，而P则用于传递一些额外的参数给Visitor.


```
public interface ElementVisitor<R, P> {

    R visit(Element e, P p);

    R visit(Element e);

    R visitPackage(PackageElement e, P p);

    R visitType(TypeElement e, P p);

    R visitVariable(VariableElement e, P p);

    R visitExecutable(ExecutableElement e, P p);

    R visitTypeParameter(TypeParameterElement e, P p);

    R visitUnknown(Element e, P p);
}
```
在ElementVisitor中定义了多个visit接口，每个接口处理一种元素类型，这就是典型的访问者模式。我们知道，一个类元素和函数元素是完全不一样的，他们的结构不一样，因此，编译器对他们的操作肯定也是不一样，通过访问者模式正好可以解决数据结构与数据操作分离的问题，避免某些操作“污染”数据对象类。

与经典访问者不同的是，ElementVisitor预留了一个visitUnknown函数来应对元素结构的变化。


首先，编译器将代码抽象成一个代码元素的树，然后在编译时对整棵树进行遍历访问，每个元素都有一个accept函数接收访问者的访问，每个访问者中都有对应的visit函数，例如visitType函数就是对类型元素的访问，在每个visit函数中对不同的类型进行不同的处理，这样就达到了差异处理的效果，同时将数据结构与数据操作分离，使得每个类型的职责单一，易于维护升级。JDK还特意预留了visitUnknow接口来应对Java语言后续发展可能添加元素类型的问题，灵活地将访问者模式的额缺点化解。



## 小结
- 优点：
1. 各角色职责分离，符合单一职责原则
2. 具有优越的扩展性
3. 使得数据结构和操作于数据结构上的操作解耦，使得操作集合可以独立变化。
4. 灵活性

- 缺点：
1. 具体元素对访问者公布细节，违反了迪米特原则
2. 具体元素变更时导致修改成本大
3. 违反了依赖倒置原则，为了达到“区别对待”而依赖了具体类，没有抽象依赖。


