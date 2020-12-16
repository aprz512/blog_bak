---
title: 000_Android单元测试系列文章介绍
index_img: /cover/16.jpg
banner_img: /cover/top.jpg
date: 2020-3-16
tags: Android-单元测试
---



最近主要在看关于代码风格与设计方面的书，它们都提到了TDD，然而从业这么多年，我却没有用到过。不得不说有点沮丧，所谓知耻而后勇，所以我准备看一下单元测试的系列教程，然后在以后的工作中慢慢实现TDD。下面的这个系列是最符合我口味的文章，所以我决定将链接搬过来。

以下这个片段是最能触动我学习TDD的，先贴在这里，如果你也觉得说的不错，不妨再深入看看。



> ## 我为什么写单元测试
>
> ### 首先，是因为我不够自信
>
> 我相信大家都有接手，或者说参与到一个新项目的经历，也许是因为换了工作，也许是因为职位调动，或其他原因。当我拿到一个新项目的时候，会有一种诚惶诚恐的感觉，因为一时间比较难理清楚整个app的结构是怎么划分的，各部分各模块之间又是什么样的关系。我怕我改了某一个地方，结果其他一个莫名其妙的地方的受到了影响，然后导致了一个bug。这对于用户群大的app，尤其严重。所以，那种时候就会希望，如果我改了某个地方，能有个东西告诉我，这个改动影响到哪些地方，这样改是不是有问题的，会不会导致bug。虽然我可以把app启动起来，看看是不是能正常工作，然而一种case能工作，并不代表所有影响到的case都能工作。尤其是在不知道有哪些地方用到了的情况下，我更加难以去遍历所有用到的地方，一个一个去验证这个改动有没有问题。哪怕我知道所有的case，这也是一个很痛苦很费时间的过程，而且很多的外部条件也很难满足，比如说需要什么样的网络条件，需要用户是会员等等。
> 在这种情况下，单元测试是才是最好的工具。首先，单元测试只是针对一个代码单元写的测试，保证一个代码单元的正确性总比保证整个app的正确性容易吧？遍历一个方法的所有参数和输出情况总比遍历一个app的所有用户场景容易吧？跑一次单元测试总比运行一次app快吧？
> 因此，在改现有的代码之前，我会先对要改的代码单元做好隔离，写好测试，再去改，改好以后跑一边单元测试，验证他们依然是通过的，这时候我才有信心，将代码合并进去。
> 同样的情况会发生在重构的时候，我是一个对烂代码不大有忍受能力的人，看到不好的代码，我会忍不住想要去重构，不然的话，没有办法写新的代码。而重构就会有风险。因为我不够自信，重构的时候，也会有一种诚惶诚恐的感觉。这时候如果有完备的单元测试的话，我就能知道我的这次重构到底破坏了哪些地方，是不是对的，这样相对来说，就会放心的多了。
> 因此，想用单元测试来保证代码的正确性，这个是我喜欢写单元测试的重要原因之一。
>
> ### 再次，是因为我没有耐心
>
> 对于有一定经验，有一定代码思想的人来说，当他拿到一个新的需求，他会先想想代码的结构，应该有那些类，那些组件，什么责任应该划分到哪里去，然后才开始动手写代码，这个是很自然的一个思维过程。然而在不写单元测试的情况下，我们可能要把整个feature都做完整，从model到controller(或Presenter、ViewModel）到view到util等等，一整套流程做下来，到最后才可能运行起来看看是不是对的，有的时候哪怕所有代码都写完了，也不一定能验证是不是对的，比如说后台还没有ready等等。总之，在没有单元测试的情况下，我们需要等到最后一刻才能手动验证代码是不是对的，然后发现原来这里错了一点，那里少了一点，然后一遍一遍的把app运行起来，改一点运行一遍。。。
> 当我开始写单元测试之后，我发现这个过程实在是太漫长了，我喜欢写完一部分功能独立的代码，就能立刻看到他们是不是正确的。如果不是的话，我可以立刻就改正，而不用等到所有代码都写完整。要达到这点，那就只有写单元测试了。
> 当然，哪怕有单元测试，最后还是要做一遍手动测试工作，然而因为前面我已经保证每一个单元都是对的，最后只不过是验证每一部分都是正确的串联起来了而已，这点相对来说，是很容易的。所以最后所需要的手动测试，可以少很多，顺利很多，也简单得多。
>
> ### 最后，是因为我懒
>
> 如前所述，如果没有单元测试的话，那就只有手工测试，把app运行起来，如果有错的话，改一点东西，再运行起来。。。这个过程太漫长太痛苦，对于一个很懒的人来说，如果能写代码来代替手工测试，每次写完代码只需要按一次快捷键，就可以直接在IDE里面看到结果，那是多爽的一件事！所以冲着这点，我也不想回头。
> 我记得上一次使用“把app运行起来”这种开发方式，还是因为调试一个动画效果。因为动画效果是很难单元测试的，那就只有改一点代码，跑一边app，觉得不对，再改一点，跑一边，这样来来回回反反复复，那感觉真是。。。



> ## 单元测试给我带来了什么
>
> ### 更好的设计
>
> 当你为自己的代码写单元测试的时候，尤其是采用[TDD](https://en.wikipedia.org/wiki/Test-driven_development)的方式，你会很自觉地把每个类写的比较小，功能单一，这是软件设计里面很重要的[SRP](https://en.wikipedia.org/wiki/Single_responsibility_principle)原则。此外，你能把每个功能职责分配的很清楚，而不是把一堆代码都塞到一个类里面（比如Activity）。你会不自觉的更偏向于采用组合，而不是继承的方式去写代码。这些都是很好的一些代码实践。
> 至于为什么TDD能够改善代码的设计，网上有很多的文章去分析和论证这个结论。我看到比较印象深刻的一句话是（具体在哪看的搜不出来了）：当你TDD的时候，你是从一开始，就从一个代码的使用者，或者说维护者的角度，去写你的代码。这样写出来的代码，自然会有更好的设计。

其实仅凭这一点，我就有足够的理由去学习TDD，在我优化自己的代码的过程中，无论如何处理，都会感觉自己处于一个瓶颈状态。而TDD刚好可以让我从使用者的角度来看待自己的代码，这是一个非常好的优化角度。



[Android单元测试（一）：首先，从是什么开始](https://chriszou.com/2016/04/13/android-unit-testing-start-from-what.html)

[Android单元测试（二）：再来谈谈为什么](https://chriszou.com/2016/04/16/android-unit-testing-about-why.html)

[Android单元测试（三）：JUnit单元测试框架的使用](https://chriszou.com/2016/04/18/android-unit-testing-junit.html)

[Android单元测试在蘑菇街支付金融部门的实践](https://chriszou.com/2016/04/25/android-unit-testing-wechat-group-share.html)



再往后面学习，可能需要[Dragger2](https://github.com/google/dagger)、[Robolectric](http://robolectric.org/)、[Mockito](http://mockito.org/)相关的知识，后面的文章也会讲到，不用太过担心，但是还是建议系列学习一下。

Mockito 的 kotlin 版本，[MockK](https://mockk.io/) 系列文章：

[MockK：一款強大的 Kotlin Mocking Library (Part 1 / 4)]([https://medium.com/joe-tsai/mockk-%E4%B8%80%E6%AC%BE%E5%BC%B7%E5%A4%A7%E7%9A%84-kotlin-mocking-library-part-1-4-39a85e42b8](https://medium.com/joe-tsai/mockk-一款強大的-kotlin-mocking-library-part-1-4-39a85e42b8))

[MockK：一款強大的 Kotlin Mocking Library (Part 2 / 4)]([https://medium.com/joe-tsai/mockk-%E4%B8%80%E6%AC%BE%E5%BC%B7%E5%A4%A7%E7%9A%84-kotlin-mocking-library-part-2-4-4be059331110](https://medium.com/joe-tsai/mockk-一款強大的-kotlin-mocking-library-part-2-4-4be059331110))

[MockK：一款強大的 Kotlin Mocking Library (Part 3 / 4)]([https://medium.com/joe-tsai/mockk-%E4%B8%80%E6%AC%BE%E5%BC%B7%E5%A4%A7%E7%9A%84-kotlin-mocking-library-part-3-4-79b40fb73964](https://medium.com/joe-tsai/mockk-一款強大的-kotlin-mocking-library-part-3-4-79b40fb73964))

[MockK：一款強大的 Kotlin Mocking Library (Part 4 / 4)]([https://medium.com/joe-tsai/mockk-%E4%B8%80%E6%AC%BE%E5%BC%B7%E5%A4%A7%E7%9A%84-kotlin-mocking-library-part-4-4-f82443848a3a](https://medium.com/joe-tsai/mockk-一款強大的-kotlin-mocking-library-part-4-4-f82443848a3a))



学完这个系列，对 Mock 应该就有自己的理解了。继续单元测试系列。

[Android单元测试（四）：Mock以及Mockito的使用](https://chriszou.com/2016/04/29/android-unit-testing-mockito.html)

[Android单元测试（五）：依赖注入，将mock方便的用起来](https://chriszou.com/2016/05/06/android-unit-testing-di.html)

[Android单元测试（六）：使用dagger2来做依赖注入，以及在单元测试中的应用](https://chriszou.com/2016/05/10/android-unit-testing-di-dagger.html)

[Android单元测试（七）：Robolectric，在JVM上调用安卓的类](https://chriszou.com/2016/06/05/robolectric-android-on-jvm.html)



看完了上面7篇文章，就可以写一些基本的测试了。但是还有一些高级用法可能不知道，下面开始介绍。

[Android单元测试（八）：Junit Rule的使用](https://chriszou.com/2016/07/09/junit-rule.html)

[Android单元测试（九）：使用Mockito Annotation快速创建Mock](https://chriszou.com/2016/07/16/mockito-annotation.html)

