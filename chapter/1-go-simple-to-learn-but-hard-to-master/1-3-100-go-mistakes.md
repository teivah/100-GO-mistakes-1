## 1.3 100个Go错误

为什么你应该阅读一本关于Go常见错误的书？为什么不通过一本可以挖掘不同主题的普通书来加深我们的知识呢？

神经科学家证明，面对错误是大脑成长的最佳时机[2]。您是否已经经历过从错误中吸取 教训并在与之相关的背景后几个月甚至几年后回忆的过程？正如珍妮特·梅特卡夫在《从错误中学习》中提出的，这一特征是因为错误具有促进作用。主要思 想是我们可以 不仅要记住错误，还要记住错误的上下文。这就是为什么从错误中学习如此有效的原因之一。

因此，为了加强这种促进作用，每个错误都将尽可能地伴随着现实世界的例子。这本书不仅是关 于理论的，而且是关于理论的。它旨在帮助您更好地避免错误并做出有意识的决定，因为您了解 错误背后的基本原理。

> 告诉我的我会忘记,给我看的我会记住,让我参与的我会理解。-- 本杰明 富兰克林

总的来说，所有的错误可以分为七大类：
* 错误
* 不必要的复杂性
* 可读性较弱
* 缺乏 API 便利性
* 次优组织
* 优化不足的代码
* 缺乏生产力

### 1.3.1 bugs

第一种错误，可能也是最明显的，是软件错误。2003年，Synopsys进行的一项研究估计，仅在美国，软件错误的成本就超过2万亿美元。此外，错误还可能导致更悲惨的影响。例如，我们可以提及Therac‑25等案例，这是一种由加拿大原子能有限公司(AELC)生产的放射治疗机。由于比赛条件，机器给予患者的辐射剂量比预期的要大数百倍，导致三名患者死亡。因此，软件错误不 仅仅与金钱有关，作为开发人员，我们应该记住我们的工作是多么有影响力。

本书将涵盖大量可能导致各种软件错误的案例，包括数据竞争、泄漏、逻辑错误和 其他缺陷。虽然准确的测试应该是尽早发现此类错误的一种方法，但有时我们可能 会因为时间压力或复杂性等不同因素而错过一些案例。因此，作为Go开发人员，确 保我们可以避免这些常见错误是必不可少的。

### 1.3.2 不必要的复杂性

下一类错误与不必要的复杂性有关。软件复杂性的一个重要部分来自于这样一个事实，即作为开发人员，我们努力思考想象中的未来。与其现在解决具体问题，不如构建可以解决任何未来用例的进化软件。然而，在大多数情况下，它导致的弊端多于好处，因为它会使代码库更难理解和推理。

回到Go，我们可以考虑很多用例，其中开发人员可能会为未来的需求设计抽象，例如接口或泛型。本书将讨论我们应该注意不要以不必要的复杂性损害代码库的主题。

### 1.3.3 可读性较弱

另一种错误是削弱可读性。正如RobertC.Martin在 *CleanCode:AHandbook ofAgileSoftwareCraftsmanship* 中所写，阅读时间与写作时间的比率远远超过10比1。我们中的大多数人已经开始在可读性不那么重要的单独项目上进行编程。但是，软件工程是具有时间维度的编程：确保我们仍然可以在数月、数年甚至数十年内工作和维护应用程序。

在Go中编程时，我们可能会犯许多会损害可读性的错误，例如嵌套代码、数据类型 表示、在某些情况下不使用命名结果参数。在本书中，我们将了解如何编写可读代码 并关心未来的读者（包括未来的自己）。

### 1.3.4 统一组织

无论是在处理一个新项目时，还是因为我们获得了不准确的反应，另一种类型的错误是组织我们的代码和项目的方式不是最优的和单一的。 这些问题会使项目更难推理和维护。

本书将涵盖Go中的一些常见错误。例如，如何构建项目、处理实用程序包或 `init` 函数。 总而言之，它应该可以帮助我们有效地、惯用地组织我们的代码和项目。

### 1.3.5 缺乏API便利性

犯一些常见的错误会削弱API对我们客户的便利程度是另一种错误。如果 一个API对用户不友好，它的表达能力就会降低，因此更难理解并且更容易出错。

我们可以考虑许多会影响API便利性的情况，例如过度使用任何类型，使用错误的创建模式来处理选项，或者盲目地应用面向对象编程的标准实践。因此，我们将涵盖阻止我们向用户公开方便的API的常见错误。

### 1.3.6 优化不足的代码

优化不足的代码是开发人员犯的另一种错误。这可能是由于各种原因，例如 对某些语言特征甚至基础知识缺乏了解。性能是最明显的影响之一，但不仅 如此。我们可以考虑为其他目标优化代码，例如准确性。

例如，在本书中，我们将看到一些阻止我们使浮点运算准确的常见技术。同时，例如，我们将介绍许多由于并行执行不佳、不知道如何减少分配或数据对齐的影响 而可能对性能代码产生负面影响的情况。我们将通过不同的棱镜解决优化问题。

### 1.3.7 缺乏生产力

在大多数情况下，我们在处理新项目时可以选择的最佳语言是什么？我们最有效率 的那个。熟悉一门语言的工作方式并充分利用它来充分利用它对于达到熟练程度至关重要。
在本书中，我们将介绍许多案例和具体示例，这将有助于在使用Go时提高工作 效率。例如，编写高效的测试以确保我们的代码正常工作，依靠标准库来提高效率，或者充分利用分析工具和linter。
现在，是时候深入研究这100个常见的Go错误了。
