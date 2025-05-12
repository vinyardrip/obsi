

| React                                                                                                                                                                                     | Выбор элементов | Vue                                                                                                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre>import { useState } from "react";<br><br>export default function Name() {<br><br>const [ name] = useState("John");<br><br>return <span>My name is {name}</span>;<br><br>}></pre><br> |                 | <pre><br><script setup><br><br>import {ref} from 'vue';<br><br>const name = ref('John');<br><br></script><br><br><template><br><br><span>My name is {{ name }}</span><br><br></template></pre> |


