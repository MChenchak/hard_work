Для выполнения задания реплизовывал АТД из курса по проектированию.

Наименьшее изменение, которое удавалось внести - хардкод возвращаемого значения. Например, из стека можно получить значение.
Первая реализация метода pop() может быть return 1, например. Далее это, конечно, нужно исправлять, т.к и значение
может быть любым, и количество элементов может меняться.
Для других структур по сути аналогично.

Внесениее настолько мелких изменений и завеедомо "ошибочных" кажется сомнительной идеей, ведь сразу же видно,
что они не соответствуют спецификации. И уж точно нет смысла коммитить такие изменения.
При таком потходе с одним простым методом может быть сделано 3 - 5 итераций, что кажется тратой времени.

Считаю, что нужно реализовывать спецификацию в полном объеме. Возможно не получится сразу учесть все краевые случаи. Это нормально.
Но заведомо хардкодить какие-то значения, делать фейковые объекты лишь бы сделать тест и вносимые изменения как можно меньше,
чтобы сократить время tdd-цикла, на мой взгляд не очень хорошая идея.
В конечном счете короткий цикл не самоцель.

Что же тогда является достаточно небольшим изменением, но все еще разумным? 
Это легче удалось понять в рабочем проекте. 
Небольшое изменение - изменение внутри одного модуля. Можно ограничиться и одним методом конкретного модуля, но похоже, что это не всегда оправдано.
И уже точно не стоит "насиловать" каждый отдельный метод ретёрнами с хардкодом или фейковыми объектами.

TDD заставляет еще больше подумать, прежде чем писать код, помогает формализовать спецификацию и даже увидеть
изъяны спецификации.