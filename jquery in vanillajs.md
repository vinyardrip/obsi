
## Выбор элементов

| JQuery                                                       | Выбор элементов                    | vanillajs                                                                                                 |
| ------------------------------------------------------------ | ---------------------------------- | --------------------------------------------------------------------------------------------------------- |
| $(el);                                                       |                                    | document.querySelector(el);                                                                               |
|                                                              |                                    | document.querySelectorAll(el);                                                                            |
|                                                              | Выбор элемента по Id               |                                                                                                           |
| $("#id");                                                    |                                    | document.getElementById("id");                                                                            |
| $(el).find(selector);                                        | Поиск элемента                     | el.querySelectorAll(selector);                                                                            |
| S(selector).each(function (index, el){ <br>  // code <br>}); | Цикл по каждому элементу           | let elements = document.querySelectorAll(el); elements.forEach(function (el, index){ <br>  // code<br>}); |
| jQuery(($) => { <br>  // code <br>});                        | Событие загрузки DOM-дерева        | document.addEventListener("DOMContentLoaded", {<br>  //code<br>});                                        |
| $(window).resize(() => {<br>  //code<br>});                  | Изменение ширины/высоты экрана     | window.addEventListener("resize", () => {<br>  //code<br>});                                              |
| $(el).on("eventName", () =>{<br>  //code<br>});              | Событие «on» (click, hover и т.п.) | el.addEventLister("eventName", () => {<br>  //code<br>});                                                 |
| $(window).scroll (() => {<br>  //code<br>});                 | Выполнение функции при скролле     | el.addEventListener("sroll", () => {<br>  //code<br>});                                                   |
|                                                              | Перемещение по DOM                 |                                                                                                           |
| $(el).prev();                                                | Предыдущий элемент                 | el.previousElement.Sibling;                                                                               |
| $(el).next();                                                | Следующий элемент                  | el.nextElementSibling;                                                                                    |
| $(el).parent();                                              | Родительский элемент               | el.parentElement;<br>  // or<br>el.parentNode                                                             |
|                                                              |                                    |                                                                                                           |
