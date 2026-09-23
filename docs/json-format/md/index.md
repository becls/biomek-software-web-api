# Biomek JSON Authoring Guide

Reference documentation for authoring Biomek methods in the JSON format.

The intended audience of the [Concept guides](concept-guides/index.md) is LLMs
that are generating Biomek methods. Humans who are familiar with Biomek
Software and Biomek instrument capabilities will want to start with [Method JSON
Structure](Method-JSON-Structure.md) and [Introduction to Pipetting
Steps](Introduction-to-Pipetting-Steps.md), and make use of the [Step
Reference](step-documentation/index.md).

When using an LLM to author a method, provide the LLM with this documentation,
the JSON-formatted export of the instrument settings for your Biomek instrument,
and the following reference manuals:

* D24406 Biomek i3 Instructions for Use
* B54473 Biomek i5 and i7 Instructions for Use
* B56358 Biomek Software Reference Manual

You will also need to provide your LLM with the list of labware, pipetting
techniques, liquid types, and tip types available on your system, and be sure to
instruct it not to invent additional items that are not on those lists.

LLMs consuming this documentation should begin with [Introduction for
LLMs](Introduction-for-LLMs.md).
