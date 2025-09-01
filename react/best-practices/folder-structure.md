# Folder Structure

This guide does not go into details when it comes to folder structure as it
heavily depends on the project, engineer preferences and the frameworks in
use. Have added some thoughts and will update this section when I have a
strong opinion.

## Goals

- A folder structure that is easy to navigate for other engineers and AI agents

## Horizontal vs. Vertically structured codebases

A horizontally structured codebase is a codebase that is horizontally sliced
by technical function whereas a vertically structured codebase generally is
sliced by domain instead.

I have not made up my mind on what is preferable for AI agents. I think a
horizontal codebase would lead to indexing more files and using more tokens
as it has to look through more folders to find the files to change. On one
hand this could be wasteful, on the other hand it could let the agent draw on
established patterns without more prompting.
