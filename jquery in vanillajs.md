
## Выбор элементов

| JQuery                                             | Выбор элементов                    | vanillajs                                                                                       |
| -------------------------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| $(el);                                             |                                    | document.querySelector(el);                                                                     |
|                                                    |                                    | document.querySelectorAll(el);                                                                  |
|                                                    | Выбор элемента по Id               |                                                                                                 |
| $("#id");                                          |                                    | document.getElementById("id");                                                                  |
| $(el).find(selector);                              | Поиск элемента                     | el.querySelectorAll(selector);                                                                  |
| S(selector).each(function (index, el){ // code }); | Цикл по каждому элементу           | let elements = document.querySelectorAll(el); elements.forEach(function (el, index){ // code}); |
| jQuery(($) => { // code });                        | Событие загрузки DOM-дерева        | document.addEventListener("DOMContentLoaded", {//code});                                        |
| $(window).resize(() => {//code});                  | Изменение ширины/высоты экрана     | window.addEventListener("resize", () => {//code});                                              |
| $(el).on("eventName", () =>{//code});              | Событие «on» (click, hover и т.п.) | el.addEventLister("eventName", () => {//code});                                                 |
| $(window).scroll (() => {/code});                  | Выполнение функции при скролле     | el.addEventListener("sroll", () => {//code});                                                   |
|                                                    |                                    |                                                                                                 |
