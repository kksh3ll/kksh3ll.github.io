---
layout: post
title: What is PSI file
data: 2018-04-01
categories: Java
---

## PSI file

一个PSI file 展示PSI元素（PSI树），一个单个的PSI file可能包含许多PSI树，一个PSI元素可以有子PSI元素。例如，可以使用PSI元素执行代码分析，如代码检查或目的操作。

The [`PsiElement`][PsiEle] class is the common base class for PSI elements.

## How do I get a PSI element?

 - From an action: `e.getData(LangDataKeys.PSI_ELEMENT)`. Note: if an editor is currently open and the element under caret is a reference, this will return the result of resolving the reference. This may or may not be what you need.

 - From a file by offset: `PsiFile.findElementAt()`. Note: this returns the lowest level element at the specified offset, which is normally a lexer token. Most likely you should use PsiTreeUtil.getParentOfType() to find the element you really need.

 - By iterating through a PSI file: using a [`PsiRecursiveElementWalkingVisitor`][PsiRec].

 - By resolving a reference: `PsiReference.resolve()`

 [PsiRec]: https://upsource.jetbrains.com/idea-ce/file/idea-ce-d00d8b4ae3ed33097972b8a4286b336bf4ffcfab/platform/core-api/src/com/intellij/psi/PsiRecursiveElementWalkingVisitor.java

 [PsiEle]: https://upsource.jetbrains.com/idea-ce/file/idea-ce-d00d8b4ae3ed33097972b8a4286b336bf4ffcfab/platform/core-api/src/com/intellij/psi/PsiElement.java