# Great Writer

In this article, we will implement a writing robot application.

> Through this section, you will learn the following key points of LazyLLM:
>
> - How to set a `formatter` for the model;
> - How to implement multi-input concurrency based on `warp` control flow;
> - How to use `bind` on functions to pass in parameters;

## Design Concept

In order to achieve longer text content generation, we plan to use two robots for implementation. The first robot is used to generate the table of contents outline and a brief description of each item, and the second robot is used to receive information from each item and output the corresponding content. Finally, the table of contents and the outline are combined to achieve the generation of long text.

Integrating the above ideas, we proceed with the following design:

![Great Writer](../../assets/4_great_writer.svg)

UI-Web receives requests from users and sends them to the table of contents outline generation robot, which generates titles and their descriptions. Next, each title and its description are sent to the second robot, which generates the corresponding content. The synthesis stage will merge the titles and their content and return it to the client.

## Code Implementation

Let's implement the above design concept based on LazyLLM.

### Design Prompt Words

According to the design, we need one model for drafting the outline and its description, and another model to write each outline based on the provided information. So we need to design two prompt words.

First is the prompt word for the outline generation robot:

```python
toc_prompt="""
You are now an intelligent assistant. Your task is to understand the user's input and convert the outline into a list of nested dictionaries. Each dictionary contains a `title` and a `describe`, where the `title` should clearly indicate the level using Markdown format, and the `describe` is a description and writing guide for that section.

Please generate the corresponding list of nested dictionaries based on the following user input:

Example output:
[
    {
        "title": "# Level 1 Title",
        "describe": "Please provide a detailed description of the content under this title, offering background information and core viewpoints."
    },
    {
        "title": "## Level 2 Title",
        "describe": "Please provide a detailed description of the content under this title, giving specific details and examples to support the viewpoints of the Level 1 title."
    },
    {
        "title": "### Level 3 Title",
        "describe": "Please provide a detailed description of the content under this title, deeply analyzing and providing more details and data support."
    }
]
User input is as follows:
"""
```

Then is the prompt word for the content generation robot:

``` python
completion_prompt="""
You are now an intelligent assistant. Your task is to receive a dictionary containing `title` and `describe`, and expand the writing according to the guidance in `describe`.

Input example:
{
    "title": "# Level 1 Title",
    "describe": "This is the description for writing."
}

Output:
This is the expanded content for writing.
Receive as follows:

"""

writer_prompt = {"system": completion_prompt, "user": '{"title": "{title}", "describe": "{describe}"}'}
```

### Setting the Model

First is the outline robot:

```python
outline_writer = lazyllm.TrainableModule('internlm2-chat-7b').formatter(JsonFormatter()).prompt(toc_prompt)
```

Here we continue to use the `internlm2-chat-7b` model and set the prompt word. The prompt word can refer to: [Master Painter](painting_master.md)

It is worth noting that the `formatter` is set here, specifying the use of `JsonFormatter`, which can extract json from the model's output string.

Since we designed the prompt word to require the outline robot to output a json-formatted string, it is necessary to parse the output here.

Then is the content generation robot:

```python
story_generater = warp(ppl.outline_writer.share(prompt=writer_prompt).formatter())
```

Here we use `TrainableModule`'s `share`, for more details see: [Multimodal Robot](multimodal_robot.md),
It allows the same model to be used as different robots with different prompt word templates.

### Assemble the Application

Let's assemble the above modules with control flow.

```python
with pipeline() as ppl:
    ppl.outline_writer = lazyllm.TrainableModule('internlm2-chat-7b').formatter(JsonFormatter()).prompt(toc_prompt)
    ppl.story_generater = warp(ppl.outline_writer.share(prompt=writer_prompt).formatter())
    ppl.synthesizer = (lambda *storys, outlines: "\n".join([f"{o['title']}\n{s}" for s, o in zip(storys, outlines)])) | bind(outlines=ppl.outline_writer)
```

In the above code, in addition to the commonly used `pipeline` control flow (for details see: [Master Painter](painting_master.md)),
It is necessary to pay attention to the use of `warp` this control flow (the red line in the design diagram).

```python
warp(ppl.outline_writer.share(prompt=writer_prompt).formatter())
```

It accepts any number of inputs and then sends them in parallel to the same branch. Since the number of json entries (i.e., the number of chapters) input by the outline robot in the previous step is uncertain,
And it is generally multiple different outputs. As the next level of content generation robot, each input needs to be processed, so using `warp` is very appropriate.

```python
ppl.synthesizer = (lambda *storys, outlines: "\n".join([f"{o['title']}\n{s}" for s, o in zip(storys, outlines)])) | bind(outlines=ppl.outline_writer)
```

Let's first focus on `bind` in this code (corresponding to the blue line in the design diagram). Here, `bind` is to send the outline output by the outline robot to `outlines` in the anonymous function.
At the same time, the content output in the previous step is packed into a tuple by `*storys`. The final synthesized content is the title + content of each chapter.

### Start the Application

Finally, we will wrap the control flow `ppl` in a client and start deployment ( `start()` ), and keep the client open after deployment ( `wait()` ).

```python
lazyllm.WebModule(ppl, port=23466).start().wait()
```

## Full Code

<details>
<summary>click to look up prompts and imports</summary>

```python
import lazyllm
from lazyllm import pipeline, warp, bind
from lazyllm.components.formatter import JsonFormatter

toc_prompt="""
You are now an intelligent assistant. Your task is to understand the user's input and convert the outline into a list of nested dictionaries. Each dictionary contains a `title` and a `describe`, where the `title` should clearly indicate the level using Markdown format, and the `describe` is a description and writing guide for that section.

Please generate the corresponding list of nested dictionaries based on the following user input:

Example output:
[
    {
        "title": "# Level 1 Title",
        "describe": "Please provide a detailed description of the content under this title, offering background information and core viewpoints."
    },
    {
        "title": "## Level 2 Title",
        "describe": "Please provide a detailed description of the content under this title, giving specific details and examples to support the viewpoints of the Level 1 title."
    },
    {
        "title": "### Level 3 Title",
        "describe": "Please provide a detailed description of the content under this title, deeply analyzing and providing more details and data support."
    }
]
User input is as follows:
"""

completion_prompt="""
You are now an intelligent assistant. Your task is to receive a dictionary containing `title` and `describe`, and expand the writing according to the guidance in `describe`.

Input example:
{
    "title": "# Level 1 Title",
    "describe": "This is the description for writing."
}

Output:
This is the expanded content for writing.
Receive as follows:

"""

writer_prompt = {"system": completion_prompt, "user": '{"title": {title}, "describe": {describe}}'}
```
</details>

```python
with pipeline() as ppl:
    ppl.outline_writer = lazyllm.TrainableModule('internlm2-chat-7b').formatter(JsonFormatter()).prompt(toc_prompt)
    ppl.story_generater = warp(ppl.outline_writer.share(prompt=writer_prompt).formatter())
    ppl.synthesizer = (lambda *storys, outlines: "\n".join([f"{o['title']}\n{s}" for s, o in zip(storys, outlines)])) | bind(outlines=ppl.outline_writer)
lazyllm.WebModule(ppl, port=23466).start().wait()
```