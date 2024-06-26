---
title: "谈谈大模型评估问题"
authors:
- user: clefourrier
translators:
- user: innovation64
---

# 谈谈大模型评估问题

由于我的团队在 Hugging Face上 从事评估和建立排行榜的工作，在两周前的 ICLR 2024 上，很多人想就这个话题向我请教（这让我感到非常意外，非常感谢所有对此感兴趣的人）。

感谢所有这些讨论，我意识到我视为理所当然的许多评估方面的想法并没有被广泛传播，这显然很有趣。

所以，我来分享一下这次对话的内容吧！

## 怎么评估大模型

首先，让我们对一些定义进行对齐。据我所知，目前主要有三种评估方法：自动化基准测试、使用人类作为裁判和使用模型作为裁判。每种方法都有其存在的原因、用途和局限性。

### 基准测试

自动化基准测试通常按以下方式工作：你想知道你的模型在某个方面的表现如何。这个“某个方面”可以是一个明确定义的具体任务，比如“我的模型在区分垃圾邮件和非垃圾邮件方面的表现如何？”或者是一个更抽象和通用的能力，比如“我的模型在数学方面的能力如何？”

由此，你会构建一个评估，通常由两部分组成：

- 一系列样本，作为输入提供给模型以观察输出结果，有时会与一个参考（称为黄金标准）进行比较。样本通常旨在尝试模拟你想要测试模型的内容：例如，如果你正在查看电子邮件分类，你会创建一个包含垃圾邮件和非垃圾邮件的数据集，尝试包括一些棘手的边缘情况等。对于大语言模型（LLMs），两个主要的任务是生成评估（在归一化后比较生成的文本与参考文本）或多选（比较提示后的可能续写文本的相对对数概率）。

- 一个指标，这是一种为模型计算得分的方法。例如，你的模型能多准确地分类垃圾邮件（正确分类的样本得分=1，错误分类得分=0）。

在模型训练集之外的数据上进行这种评估更有意义，因为你想要测试模型是否具有良好的泛化能力。你不想得到一个只能分类它已经“见过”的电子邮件的模型，那样的模型不会非常有用！

注意：如果一个模型只能在训练数据上预测得好（并且没有潜在地学习到更多高级别的通用模式），那么这个模型被认为是**过拟合的**。在不太极端的情况下，你仍然想要测试你的模型是否能够泛化到训练集分布之外的数据模式（例如，在只见过关于假银行的垃圾邮件之后，对关于“健康”产品的垃圾邮件进行分类）。

这种方法对于定义非常明确的任务非常有效，因为性能“容易”评估和测量：当你在字面上测试你的模型进行垃圾邮件分类时，你可以说“模型正确分类了这些样本的n%”。对于大型语言模型的基准测试，可能会出现一些问题，[例如模型在多选评估中基于呈现的顺序偏好特定的选择](https://arxiv.org/abs/2309.03882)，以及生成性评估依赖于[如果设计不当很容易导致不公平](https://huggingface.co/blog/open-llm-leaderboard-drop)的归一化，但总体来说，它们仍然在任务层面上提供了信号。

对于能力而言，很难将它们分解为明确定义和精确的任务：数学能力好意味着什么？擅长算术？逻辑？能够对数学概念进行推理？

在这种情况下，人们倾向于进行更多的“整体性”评估，而不是将能力分解为实际任务，而是假设在一般样本上的表现将是我们试图测量的内容的良好代理。例如，GSM8K 由实际的高中数学问题组成，解决这些问题需要一整套能力。这也意味着失败和成功都很难解释。一些能力或主题，例如“这个模型擅长写诗吗？”或“模型的输出有帮助吗？”甚至更难以用自动指标来评估 - 同时，模型现在似乎具有越来越多的**通用**能力，因此我们需要以更广泛的方式评估它们的能力。（例如，科学界曾经就大语言模型是[否](https://twitter.com/DimitrisPapail/status/1719119242186871275)能够[画独角兽](https://arxiv.org/abs/2303.12712)进行过辩论。目前可能还做不到，但显然这是一个值得研究的重要点。）

自动基准测试也往往有另一个问题：一旦它们以纯文本形式公开发布，它们很可能会最终（通常是意外地）出现在模型的训练数据集中。一些基准测试的创建者，比如 BigBench 的作者，试图通过添加一个“金丝雀字符串”（一个非常特定的字符组合）来缓解这个问题，供人们在训练集中寻找并删除，但并非所有人都知道这个机制，也不是所有人都在尝试进行这种删除。还有大量的基准测试，寻找所有这些测试在数据中的意外副本是昂贵的。其他选项包括以[加密形式](https://arxiv.org/pdf/2309.16575)提供基准测试，或者通过一个[门控系统](https://huggingface.co/datasets/Idavidrein/gpqa)提供。然而，当评估黑盒 API 后面的封闭模型时，无法保证提供的数据不会被内部用于训练或微调。

当评估数据集最终出现在训练集中的情况被称为污染，而被污染的模型将具有高基准性能，但这种性能并不能很好地泛化到基础任务（关于污染的详细描述可以在[这里](https://aclanthology.org/2023.findings-emnlp.722/)找到，这里有一种有趣的方式来[检测它](https://arxiv.org/abs/2311.06233)）。解决污染的一种方法是运行[动态基准测试](https://arxiv.org/abs/2104.14337)（在定期刷新的数据集上进行评估，以提供在系统性地未见的新数据上的得分），但这种方法的长期成本较高。

### 人类作为裁判评估

解决污染问题以及更开放式评估的方法之一是让人类评估模型的输出。

这通常是通过让人类首先提示模型，然后根据指导方针给模型的答案评分或对几个输出进行排名来完成的。使用人类作为裁判可以研究更复杂的任务，比自动化指标具有更大的灵活性。它还防止了大多数污染情况，因为编写的提示（希望）是新的。最后，它与人类偏好相关性很好，因为这正是被评估的内容！

在人类参与的情况下评估模型存在不同的方法。

"Vibes-checks"（氛围检查）是社区中某些成员针对未公开的提示单独进行的手动评估的名称，目的是获得模型在许多用例上表现如何的总体“感觉”，这些用例从编程到色情文学的质量都有。（我还见过使用“金丝雀测试”这个术语，这是对煤矿中高信号金丝雀方法的参考）。通常在 Twitter 和 Reddit 上分享，它们主要构成轶事证据，并且倾向于对确认偏差高度敏感（换句话说，人们倾向于找到他们所寻找的东西）。然而，有些人一直在尝试进行更系统化的氛围检查评估；例如，用户 Wolfram Ravenwolf 通过博客以非常系统化的方式分享他的模型比较发现（参见[这里](https://huggingface.co/blog/wolfram/llm-comparison-test-llama-3)的例子）。

使用社区反馈来建立大规模模型排名就是我们所说的**竞技场**。这种方法的著名例子是 [LMSYS 聊天机器人竞技场](https://huggingface.co/spaces/lmsys/chatbot-arena-leaderboard)，社区用户被要求与模型进行聊天，直到他们认为一个模型比另一个更好。然后，将这些投票汇总到一个 Elo 排名（比赛排名）中，以选出“最佳”模型。这种方法的明显问题是高度主观性——很难强制许多社区成员使用广泛指导方针进行一致的评分，尤其是由于注释者的偏好往往受到[文化限制](https://arxiv.org/abs/2404.16019v1)（例如，不同的人喜欢不同的讨论话题）。人们可以希望这种效果通过大量投票的规模来平滑，通过“群体智慧”效应（这个效应是由一位名为 Galton 的统计学家发现的，他观察到尝试估计数值答案的个体答案，比如猪的重量，可以被建模为围绕实际答案的概率分布）。

最后一种方法是**系统性注释**，你为选定的付费注释者提供极其具体的指导方针，以尽可能去除主观性偏差（这是[ScaleAI](https://scale.com/guides/data-labeling-annotation-guide#hight-quality-data-annotations) 和大多数数据注释公司使用的方法）。使用人工评估模型的方法可能会变得非常昂贵，因为你需要不断地以非自动化的方式对每个新模型进行评估。而且，这种方法可能还是会受到人们个人偏见的影响，[因为](https://arxiv.org/abs/2205.00501)不同的人可能会根据自己的身份和观点，给出非常不同关于模型回答是否有害的评价。

最近的[研究](https://arxiv.org/pdf/2309.16349)还表明，人类评估者倾向于根据第一印象来估计答案的质量，而不是实际的事实性或忠实度。众包注释者特别敏感于语调，并低估了断言性答案中的事实或逻辑错误数量。换句话说，如果一个模型以自信的语调说出错误的事情，人类评估者注意到这一点的可能性要小得多，这可能会使评分偏向于更断言性的模型。（专家注释者不太可能受到这些偏差的影响。）这种人类偏差在另一篇[论文](https://arxiv.org/pdf/2310.13548)中得到了证实：人类最有可能喜欢吸引他们观点或与他们意见或错误一致的答案，而不是事实正确的答案。

这些偏差并不出乎意料，但必须加以考虑：并非所有用例都应该依赖使用人类注释者，特别是众包的、非专家的注释者——任何需要事实性的任务（如代码编写、模型知识的评估等）都应该包括另一种更稳健的评估类型来完成基准测试。

### 模型作为裁判

为了减轻人工注释者的成本，一些人开始尝试使用模型或派生的工件（最好与人类偏好一致）来评估模型的输出。这种方法并不新鲜，因为你可以找到早在 2019 年就有的技术，用于从[模型嵌入](https://arxiv.org/abs/1904.09675)中衡量摘要质量。

有两种评分方法：使用[通用型、高能力模型](https://arxiv.org/abs/2306.05685v4)或使用专门为偏好数据训练的[小型专业模型](https://arxiv.org/pdf/2405.01535)。前者给出的结果与人类偏好相关性很好，但大多数足够强大的模型往往是闭源的，因此可能会在 API 后面发生变化，并且无法解释。

大语言模型（LLM）作为裁判有几个严重的局限性：它们在评分答案时倾向于[偏向自己的输出](https://arxiv.org/abs/2404.13076)，在[提供一致的分数范围方面表现不佳](https://twitter.com/aparnadhinak/status/1748368364395721128)（尽管你可以通过要求模型在[提供分数之前](https://twitter.com/seungonekim/status/1749289437165769177)解释其推理来改善这一点），并且实际上与[人类排名](https://arxiv.org/pdf/2308.15812)并不一致。

我个人对使用模型作为裁判的主要不满之处在于，它们在答案选择中引入了非常微妙且无法解释的偏差。我觉得，就像在遗传学研究中过度杂交一样，你最终会得到功能障碍的动物或植物，通过使用 LLM 来选择和训练 LLM，我们很可能引入微小的变化，这些变化在几代之后会产生更大的影响。我相信这种类型的偏差在小型的、更专业的模型作为裁判时不太可能发生（例如毒性分类器），但这仍然需要经过严格的测试和证明。

## 为什么需要大模型评估

现在我们已经了解了如何进行评估，那么它实际上有什么用呢？

我坚信人们进行评估主要有三个原因，这些原因往往被混淆在一起，但实际上它们非常不同，每个原因都回答了一个不同的问题。

### 1）我的模型训练得好吗？我的训练方法可靠吗？- 非回归测试

非回归测试是一个来自软件行业的概念，用来确保小的更改没有破坏整体方法。

这个想法是这样的：当你向软件添加新功能或修复代码库中的问题時，你有没有破坏其他东西？这就是非回归测试的目的：确保你的软件的预期、高级行为没有突然因为一个（看似不相关的）更改而被破坏。

当你选择一个设置来训练模型时，你想要测试非常相似的东西，并确保你的更改（选择不同的训练数据、架构、参数等）没有“破坏”这些属性模型的预期性能。

举一个具体的例子，你会期望一个 70 亿参数的基础 LLM 在训练后（在多项选择题的 MMLU 上）得分在 50 到 65 之间，另一方面，知道性能在 20 到 30 之间波动表明没有学习发生。

对于“非回归”评估，你需要查看 1）评估分数的**轨迹**（现在的性能是否比开始训练时要好），2）评估分数的**范围**（性能是否在预期范围内）。实际上...你并不关心确切的分数本身！

因此，这种评估并不是为了告诉你关于模型实际能力的任何信息，而是为了确认你的训练方法与其它训练方法一样“可靠”，并且你的模型的行为方式相似。我相信，甚至一些只看文本困惑度（概率）变化的评估就足以完成这一步，但通常你希望基准测试具有高“信噪比”，或者换句话说，你希望确保分数的大变化反映了你的模型的大变化。

### 2）哪个模型是最好的？我的模型比你的模型好吗？- 排行榜和排名

评估的下一个作用就是简单地对模型进行排序，以找到和选择总体上最好的架构和方法。如果你有一个排行榜，拿最好的模型，如果它在你的用例上不起作用，那么下一个最好的模型也不太可能起作用。在他们的[论文](https://arxiv.org/pdf/2404.02112)中，作者从 ImageNet 时代的基准测试和数据集设计经验中得出，由于分数容易受到不稳定性的影响，评估模型的唯一可靠方法是通过排名，特别是通过找到提供一致且稳定排名的广泛评估组。

我相信寻找排名稳定性确实是一个非常有趣的模型基准测试方法，因为我们已经显示出 LLM 在自动基准测试上的得分非常容易受到提示中微小变化的影响，而且人类评估的一致性也不高 - 在使用稳健的评估方法时，排名实际上更稳定。

如果分数本身并不那么重要，那么使用模型的相对排序能否告诉我们更有价值的信息呢？

在相关的 ICLR 2024 评估全体会议上，Moritz Hardt 比较了对 Open LLM 排行榜（通过微小的分数修改，完全在分数范围内）和 Chatbot 竞技场（通过添加一个糟糕的竞争者来观察它如何影响 Elo 排名）添加干扰的情况。目前，这些基准测试并没有提供稳定和一致的排名。我们将在未来版本的 Open LLM 排行榜中确保探索这个方面！

### 3)在模型能力的领域，我们现在处于什么位置？我的模型能做X吗？

“你如何知道模型能否做 X？”是一个经常出现的问题，我认为这是一个非常合理的问题。

然而，对于任何复杂的能力，**我们现在不能简单地说是“这个模型在这方面是最好的”，而应该说“这个模型在这个我们希望是一个好的代理任务的方面是最好的，但我们无法保证”**。

我们严重缺乏对机器学习模型能力的好定义和框架，特别是对于那些围绕推理和心智理论的能力。然而，这并不仅限于机器学习！在人类和动物研究中，定义什么是“能力”也是相当困难的，而且尝试提供精确分数的指标（例如智商和情商）也备受争议和讨论，这是有原因的。

我们可能需要借鉴社会科学来思考能力的评估，因为这些领域的人们习惯于认真考虑数据收集和分析中的混杂因素。然而，我也相信以下两点很可能成立：1）我们根本无法定义这些广泛的能力，因为我们目前无法定义人类和动物的能力；2）以人类（或动物）为参照构建的框架可能不会很好地转移到模型上，因为底层的行为和假设并不相同。

## 结论

LLM 评估现在通常以以下方式进行：使用自动化基准测试，可能会受到污染和缺乏“通用性”（后者不一定是一件坏事，因为专业评估是有趣的）的影响；使用人类评估，在小规模上可能缺乏可重复性，并且总体上可能受到心理偏见的影响（例如对奉承答案的偏好），尽管我们可以在大规模上希望一些偏见会被平滑处理；使用模型作为裁判，评估时存在非常微妙的偏见，可能不容易被发现，但可能会引入下游的扰动。

然而，一切并非全无希望：在限制范围内，评估仍然能够提供一些信号，让我们了解哪些新的训练方法或数据集听起来有前景，这既可以通过查看性能是否在预期范围内（非回归测试），也可以通过查看模型总体排名（足够稳定的评估）来观察。我们还可以希望，跨主题和任务收集足够的数据点将为我们提供足够的信号，以了解整体模型性能，但不要假设关于更“通用”的能力。

与炒作相反，我们目前无法真正评估“通用模型能力”，首先因为我们没有定义这意味着什么。然而，作为研究领域的LLM 评估目前还处于初级阶段，还有很多工作要做，这非常令人兴奋！我们可以从许多领域汲取灵感，从机器学习可[解释性](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html)到社会学，以定义新的指标和任务。跨学科工作可能会为该领域打开非常新颖的酷炫方向！

## 致谢

非常感谢所有在会议中对我们讨论评估感兴趣的人，包括但不限于来自 Scale AI 的 Summer Yue、来自 Max Planck Institute 的 Moritz Hardt、来自 Allen AI 的 Luca Soldaini 和 Ian Magnusson、来自 Anthropic 的 Ludwig Schmidt、来自 Cohere 的 Max Bartolo、来自 Liquid AI 的 Maxime Labonne、来自 Meta 的 François Charton、来自 UK AI Safety Institute 的 Alan Cooney 和来自 Together AI 的 Max Ryabinin。

也非常感谢来自 Hugging Face 的 Yacine Jernite 和 Irene Solaiman 对他们在这份文件上的宝贵反馈。

最后但同样重要的是，感谢 Hugging Face 的评估和排行榜团队，特别是 Nathan Habib，我们共同进行的讨论和工作！
