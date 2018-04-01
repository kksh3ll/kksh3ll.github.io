---
layout: post
title: What is PSI file
data: 2018-03-31
categories: Java
---

## PSI file

The Program Structure Interface, 通常被称为是PSI，在IntelliJ平台上，负责解析文件，创建句法和语义编码模型，PSI集合了平台的许多特性。

A `PSI (Program Structure Interface)` file is the root of a structure representing the contents of a file as a hierarchy of elements in a particular programming language.

PsiFile 类 是所有PSI文件的共同基类，例如 PsiJavaFile 类 是Java PSI文件的基类

## How do I get a PSI file

 - From an action: e`.getData(LangDataKeys.PSI_FILE)`.

 - From a VirtualFile: `PsiManager.getInstance(project).findFile()`

 - From a Document: `PsiDocumentManager.getInstance(project).getPsiFile()`

 - From an element inside the file: `psiElement.getContainingFile()`

 - 用指定的name在项目project中查找: `FilenameIndex.getFilesByName(project, name, scope)`

## What can I do with a PSI file

Most interesting modification operations are performed on the level of individual PSI elements, not files as a whole.

To iterate over the elements in a file, use `psiFile.accept(new PsiRecursiveElementWalkingVisitor()...)`;

## Where does a PSI file come from

As PSI is language-dependent, PSI files are created through the Language object, by using the `LanguageParserDefinitions.INSTANCE.forLanguage(language).createFile(fileViewProvider)` method.

Like documents, PSI files are created on demand when the PSI is accessed for a particular file.

## 如何创建PSI file

The [`PsiFileFactory.getInstance(project).createFileFromText()`][PsiFile] method creates an in-memory PSI file with the specified contents.

To save the PSI file to disk, use the `PsiDirectory.add()` method.

## What are the rules for working with PSI

Any changes done to the content of PSI files are reflected in documents, so all rules for working with documents (read/write actions, commands, read-only status handling) are in effect。

[PsiFile]: https://upsource.jetbrains.com/idea-ce/file/idea-ce-d00d8b4ae3ed33097972b8a4286b336bf4ffcfab/platform/core-api/src/com/intellij/psi/PsiFileFactory.java