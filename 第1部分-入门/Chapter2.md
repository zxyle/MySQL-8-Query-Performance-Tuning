# 查询调优方法

有几种解决问题的方法。 在极端情况下，您可以先潜水，然后尝试进行一些更改。尽管这似乎可以节省时间，但通常会造成挫败感，即使所做的更改似乎奏效，您也无法确定自己是否真正解决了根本问题，或者只是暂时性地改善了问题。

相反，建议是通过进行分析并使用监视来确认更改的效果，从而在方法上进行工作。 本章将向您介绍一种方法，该方法在解决MySQL问题时会很有用，重点是性能调优。 首先介绍该方法中的步骤。 然后，本章的其余部分将更详细地讨论每个步骤，以及为什么花尽可能多的时间主动地进行工作很重要。

------

**注意**：此处描述的方法基于Oracle支持中用于解决客户报告的问题的方法。

------



## 总览

MySQL性能调优可以看作是一个永无止境的过程，在此过程中，使用迭代方法逐步改善性能。 显然，有时会出现诸如查询之类的特定问题，需要花费半小时才能完成，但是请记住性能不是二进制状态，这一点很重要，因此有必要知道足够好的性能。 否则，您将永远无法完成单个任务。

图2-1显示了如何描述性能调优生命周期的示例。 该循环从左上角开始，包括四个阶段，其中第一个阶段是验证问题。

![](../附图/Figure%202-1.png)

当您遇到性能问题时，第一步是验证问题所在，包括收集问题证据并定义考虑解决问题的要求。

第二阶段涉及确定性能问题的原因，而第三阶段则确定解决方案。 最后，在第四阶段中实施解决方案。 解决方案的实施应包括验证更改后的效果。

------

**提示**：在危机期间进行救火和主动工作时，此循环均有效。

------

然后，您就可以从头做起，要么进行第二次迭代，以进一步提高刚刚关注的问题的运行性能，要么可能需要解决第二个问题。也可能在周期之间会有一段很长的时间。

然后，您可以准备从头开始，或者进行第二次迭代以进一步改善刚遇到性能的问题，或者您可能需要处理第二个问题。 周期之间也可能会有很长的一段时间。

## 验证问题

在尝试确定导致问题的原因和解决方案之前，必须清楚您试图解决什么问题。 仅仅说“ MySQL慢”是不够的–这意味着什么？ 一个具体的问题可能是“前网页第二部分中使用的查询需要五秒钟”或“ MySQL每秒只能维持5000个事务”。 您越具体，解决问题的机会就越大。

问题的定义还应该包括验证问题所在。 最初看起来是什么问题与真正的问题之间可能会有区别。 验证问题可能很简单，例如执行查询并观察查询是否真的花了那么长的时间，或者可能涉及查看监视。

准备工作还应包括从监视中收集基准或运行说明问题的数据集。 没有基准，您可能无法在故障排除结束时证明已解决了该问题。

最后，您需要确定性能调优的目标是什么。引用斯蒂芬 · 科维的《高效人士的习惯》

*开始就要考虑如何结束。*

慢查询的运行速度的最低可接受目标是什么？或者所需的最小事务吞吐量是多少？这将确保您知道在进行更改时是否达到了目标。

问题已明确定义和验证后，可以开始分析问题并确定原因。

## 确定原因

第二阶段是确定性能不佳的原因。确保你思路开放，并考虑整个堆栈，所以你最终不会盯着自己的盲目的一个方面，事实证明，没有任何与问题有关。

第二阶段是确定性能低下的原因。 确保心胸开阔，并考虑整个堆栈，以免最终看不到与问题无关的一个方面。

当你认为你知道原因时，你还需要争辩为什么这就是原因。EXPLAIN 语句的输出清楚地显示查询执行完整表扫描，因此这可能是原因，或者您可能有一个图形显示 InnoDB 重做日志已满 75%，因此您可能会出现异步刷新导致临时性能问题。

当您认为自己知道原因时，还需要争论为什么这就是原因。 您可能具有EXPLAIN语句的输出，清楚地表明查询执行了全表扫描，因此这可能是原因，或者您可能有一个图显示InnoDB重做日志已满75％，因此您可能具有异步 刷新会导致临时性能问题。

找出原因往往是调查最难的部分。一旦知道原因，您就可以决定解决方案。

## 确定解决方案

确定要调查的问题的解决方案需要两个步骤。 第一步是寻找可能的解决方案； 其次，您必须选择实施哪一个。

当你寻找可能的解决方案时，进行头脑风暴会很有用，你写下所有你能够想到的想法。重要的是，不要将自己限制在根本原因所在的狭窄区域，因为在尽可能多的地方可能找到其他区域的解决方案。例如，由于上一章中提到的内存碎片而导致的停滞，其中解决方案是更改 MySQL 的配置，以使用直接 I/O 来减少操作系统 I/O 缓存的使用。您还应同时考虑短期解决方法和长期解决方案，因为如果需要重新启动或升级 MySQL、更改硬件或类似解决方案，则可能并不总是能够立即实施完整解决方案。

当您寻找可能的解决方案时，进行头脑风暴，写下所有可以想到的想法，可能会很有用。 重要的是，不要将自己限制在根本原因所在的狭窄区域，因为在尽可能多的地方可能找到其他区域的解决方案。 上一章提到的由于内存碎片化导致的停顿就是一个例子，解决方案是更改MySQL的配置以使用直接I/O以减少对操作系统I/O缓存的使用。 您还应该牢记短期解决方法和长期解决方案，因为如果需要重新启动或升级MySQL，更改硬件或类似功能，则不一定总是可以立即实施完整的解决方案。

------

**提示** 有时被低估的解决方案是升级 MySQL 或操作系统以使用新功能。但是，当然，您需要进行小心测试，以验证您的应用程序是否与新版本兼容，并特别小心优化器是否有任何更改会导致查询性能不佳。

------

确定解决方案的第二部分是选择最适合的候选解决方案。为了做到这一点，你必须争论每个解决方案为什么它的工作原理，什么是利弊。在这一步中，重要的是要对自己诚实，并小心地考虑可能的副作用。

一旦您充分了解所有可能的解决方案，您可以选择继续使用哪种解决方案。您还可以选择一个解决方案作为临时缓解措施，同时使用更可靠的解决方案。在这两种情况下，下一阶段是实现解决方案。

## 实施解决方案

您可以通过一系列步骤来实现该解决方案，其中定义了行动计划，测试行动计划，优化行动计划等，直到最终将该解决方案应用于生产系统。 重要的是不要急于执行此过程，因为这是发现解决方案问题的最后机会。 在某些情况下，测试还可能表明您将需要放弃该解决方案，然后返回上一个阶段并选择其他解决方案。 图2-2说明了实施解决方案的工作流程。

![](../附图/Figure%202-2.png)

您采用选择的解决方案，并为其创建一个行动计划。 在这里，非常具体很重要，这样可以确保测试的行动计划也是最终在生产系统上应用的行动计划。 写下将要使用的确切命令和语句很有用，这样您就可以复制和粘贴它们，或者将它们收集在脚本中，以便可以自动应用它们。

然后，您需要在测试系统上测试操作计划。 重要的是要尽可能地反映生产环境。 您在测试系统上拥有的数据必须代表您的生产环境数据。 实现此目的的一种方法是复制生产环境数据，可以选择使用数据屏蔽来避免将敏感信息（例如个人详细信息和信用卡信息）复制到生产系统之外。

------

**提示** MySQL企业版订阅（收费订阅）包括数据屏蔽功能：www.mysql.com/products/enterprise/masking.html。

------

测试应验证该解决方案已解决问题，并且没有意外的副作用。 要求什么测试取决于您要解决的问题和建议的解决方案。 如果查询速度慢，则需要在实施解决方案后测试查询的性能。 如果修改一个或多个表上的索引，则还必须验证它如何影响其他查询。 实施解决方案后，您可能还需要对系统进行基准测试。 在所有情况下，您都需要与问题验证期间收集的基准进行比较。

第一次尝试可能无法按预期进行。 通常，仅需要对行动计划进行一些细化，有时您可能不得不完全放弃提议的解决方案，然后返回上一个阶段并选择另一个解决方案。 如果建议的解决方案部分解决了问题，则您也可以选择将其应用于生产系统，然后重新开始并评估如何继续改善性能。

如果您对测试显示解决方案有效感到满意，则可以将其应用于暂存系统，如果一切仍在工作，则可以将其应用于生产系统。 完成此操作后，您再次需要验证它是否有效。 无论您多么谨慎地设置代表生产系统的测试系统，由于某种原因，该解决方案都可能无法按预期在生产中完全正常工作。 本书作者遇到的一种可能性是，本质上随机的索引统计信息是不同的，因此在将解决方案应用于生产系统时，必须使用ANALYZE TABLE语句来更新索引统计信息。

如果该解决方案有效，则应收集新的基准，以用于将来的监视和优化。 如果该解决方案不起作用，则需要通过回滚更改并寻找新的解决方案或进行新一轮的故障排除并确定为什么该解决方案不起作用并应用第二个解决方案来决定如何继续。

## 积极工作

性能调优是一个永无止境的过程。 如果您的系统从根本上来说是健康的，那么大部分工作都将在预防紧急情况和紧急程度相对较低的地方积极开展。 这不会给您的工作带来太多关注，但是它将使您的日常生活压力减轻，并且用户会更快乐。

------

**注意**：这一讨论在某种程度上基于斯蒂芬·科维的《高效人士7个习惯》中的习惯3"*把第一件事放在第一位*"。

------

图2-3显示了如何将任务分类为紧急程度和重要性。 紧急任务通常会引起其他人的注意，而其他任务可能很重要，但是只有在未及时完成的情况下它们才会变得可见，因此它们突然变得紧急。

![](../附图/Figure%202-3.png)

最容易归类的任务是与危机相关的任务，例如生产系统故障和公司损失收入，因为客户无法使用产品或进行购买。 这些任务既紧急又重要。 在这些任务上花费大量时间可能会让您感到很重要，但这也是一种非常紧张的工作方式。

解决性能问题的最有效方法是处理重要但不是紧迫的问题。 这是预防危机发生的前瞻性工作，它包括监视，在问题变得明显之前进行改进等等。 此类别中的一项重要任务也是准备，因此您已准备好应对危机。 例如，这可能是为了建立一个备用系统，以备您在发生危机时进行故障转移或快速启动替换实例的过程。 这可以帮助减少危机的持续时间，并使危机重新回到重要但并非紧急的类别。 您花费在此类任务上的时间越多，通常您就越成功。

最后两类包括不太重要的任务。 紧急但不重要的任务包括无法重新安排的会议，其他人推动的任务以及可感知的（但不是真实的）危机。 非紧急和不重要的任务包括管理任务和检查电子邮件。 当然，其中一些任务可能对于保持您的工作是必需的并且很重要，但是对于保持MySQL的良好性能并不重要。 尽管总是必须处理这些类别中的任务，但重要的是要尽量减少在此花费的时间。

避免处理不重要的任务的一部分包括您了解任务的重要性，例如，通过定义性能何时足够好，这样就不会最终对查询或吞吐量进行过度优化。 在实践中，如果非重要任务吸引了组织中其他人的注意力，这些任务当然会很困难（这些任务通常是紧急任务），但是重要的是，您应尽可能多地尝试 可以将工作转移到重要但非紧急的任务上，以避免危机任务在以后的时间接管。

## 总结

本章讨论了可用于解决MySQL性能问题（以及其他类型的问题）的方法，以及主动工作的重要性。

报告问题后，您便可以开始验证问题所在并确定已解决的问题。 对于天生具有开放性的绩效问题，重要的是要知道什么是足够好的，否则您将冒着永不停止执行危机管理并回到主动工作的风险。

一旦有了清晰的问题描述，就可以确定原因。 原因明确之后，您可以确定要解决的问题。 最后一个阶段是实施解决方案，如果事实证明您首先选择的解决方案不起作用或具有不可接受的副作用，则可能需要您重新考虑潜在的解决方案。 在这种情况下，重要的是要尽可能实际地测试解决方案。

本章的最后部分讨论了花尽可能多的时间进行积极主动的工作的重要性，这些工作可以防止发生危机，并在危机发生时帮助您做好准备。 这将帮助您减轻工作压力，并以更好的状态管理数据库。

正如本章所讨论的，在将解决方案部署到生产系统之前测试其影响非常重要。 下一章将介绍基准测试，重点是Sysbench基准测试。