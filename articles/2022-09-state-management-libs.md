---
title: "状態管理ライブラリーメモ"
type: "tech"
emoji: "🚉"
topics: ["JavaScript", "状態管理"]
published: true
---

前回の[svelte-store-treeというライブラリーをリリースしました](https://zenn.dev/igrep/articles/2022-09-svelte-store-tree)と言う記事を書く前、「Svelteを問わず、他の状態管理ライブラリーをあれこれ調べてまとめて比較しよう」と思って調べたときのメモです。結局、いずれも結構コンセプトに差があり、素直に比較するのは難しそうなので前回の記事に載せるのは諦めましたが。事実上ただのリンク集ですが、何かの参考になれば幸いです。

# 調査方法

- [React ステート管理 比較考察 - uhyo/blog](https://blog.uhy.ooo/entry/2021-07-24/react-state-management/)から著名なライブラリーを
- [Vue 3 State Management Guide - Made with Vue.js](https://madewithvuejs.com/blog/vue3-state-management-guide)

- 以下のnpmの検索結果からそれらしいパッケージを集めてドキュメントを読む
    - <https://www.npmjs.com/search?q=keywords:state>
    - ... が、やはり件数が多すぎるのである程度列挙したところで止めた

# [Redux](https://www.npmjs.com/package/redux)

読んだドキュメント:

- [Redux Fundamentals, Part 1: Redux Overview | Redux](https://redux.js.org/tutorials/fundamentals/part-1-overview)

# [Recoil](https://recoiljs.org/)

[Atoms | Recoil](https://recoiljs.org/docs/basic-tutorial/atoms/)

[Selectors | Recoil](https://recoiljs.org/docs/basic-tutorial/selectors)

# [Jotai](https://jotai.org/)

[Jotai, primitive and flexible state management for React](https://jotai.org/)

[Comparison — Jotai, primitive and flexible state management for React](https://jotai.org/docs/basics/comparison)

# MobX

[README · MobX](https://mobx.js.org/README.html)

[The gist of MobX · MobX](https://mobx.js.org/the-gist-of-mobx.html)

# [Pinia](https://pinia.vuejs.org/)

[Introduction | Pinia](https://pinia.vuejs.org/introduction.html)

[State | Pinia](https://pinia.vuejs.org/core-concepts/state.html)

[Defining a Store | Pinia](https://pinia.vuejs.org/core-concepts/)

[Getters | Pinia](https://pinia.vuejs.org/core-concepts/getters.html)

[Actions | Pinia](https://pinia.vuejs.org/core-concepts/actions.html)

# [Zustand](https://www.npmjs.com/package/zustand)

# SolidJSのstore

- [チュートリアル](https://www.solidjs.com/tutorial/stores_createstore)
- [詳細なドキュメント](https://www.solidjs.com/docs/latest/api#%E3%82%B9%E3%83%88%E3%82%A2%E3%81%AE%E4%BD%BF%E7%94%A8)
- [ソースコード](https://github.com/solidjs/solid/blob/main/packages/solid/store/src/store.ts)

# 所感

有名そうなライブラリーをとりあえず集めでドキュメントを軽く読んだ。どうやら、例えばRecoilのselectorがsvelte-store-treeと概ね同じことを実現している模様
