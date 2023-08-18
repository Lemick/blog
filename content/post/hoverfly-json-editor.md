+++
title = "Simulation Editor UI for Hoverfly"
description = "Simulation Editor UI for Hoverfly"

date = "2023-04-20"
tags = [
    "hoverfly",
    "test",
]
+++

The source code is available [here](https://github.com/Lemick/hoverfly-ui)

As a developer, I understand the importance of having effective testing tools to improve the quality of software products. One such tool is Hoverfly, a popular open-source simulation tool used to create HTTP(S) services that mimic the behavior of real APIs.

With Hoverfly, developers can test their code against mocked services that mimic the behavior of real APIs, enabling them to detect and fix issues before the code goes into production. However, working with Hoverfly simulations in JSON format can be challenging, especially as the file grows in size. I developed a Hoverfly simulation editor UI to make it easier for developers to create and manipulate simulations.

While there are already tools available for working with Hoverfly simulations, such as [the one developed by SpectroLabs](https://github.com/MegafonWebLab/hoverfly-ui) (which I have not tested), I needed a tool that could take JSON input. This inspired me to develop my own Hoverfly simulation editor UI, which could provide a more user-friendly interface for manipulating and creating simulations.

## The tool

The UI allows developers to create and manipulate simulations with ease. With this tool, they can use a simple UI to create and edit their simulations, rather than having to modify a JSON file manually. This can save a significant amount of time and make the simulation creation process much more efficient.

[You can give it a try here](https://lemick.github.io/hoverfly-ui/)

## Areas for improvement

While my Hoverfly simulation editor UI has been helpful in my work, I have also identified some improvement. For example, some matchers are missing from the UI, which can limit the tool's functionality. Additionally, the user experience could be improved to make the tool more intuitive and user-friendly. I also need to test my tool with different versions of Hoverfly to ensure that it works well with all versions. I think it could be also powerful as an IntelliJ plugin.