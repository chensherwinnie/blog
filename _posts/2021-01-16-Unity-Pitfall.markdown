---
title:  "Unity- Pitfall"
date:   2021-07-28
categories: Unity
permalink: "blog/unity/:title"
tags: C# learning
---

### CanvasGroup.interacterable
`CanvasGroup.interactable` does not affect children's `interactable` variable directly. Instead, use `Selectable.IsInteractive()` to check if the component is truly interactable.
