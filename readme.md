# Движок для создания танков на js

## API танков

Приложение зациклино через setInterval(tick, 200). Каждый тик выполняет: ai всех ботов; пересчёт положений снарядов и ботов, включая коллизии, попадания, подбор поверапов и смерти.

AI - это js-функция, вызванная в контексте инстанса бота.

### Методы
```js
// Направления
this.left();
this.up();
this.right();
this.down();
```
При вызове любого метода направления танк поворачивается в соответствующую сторону и проезжает в эту сторону 1 условный пиксель. Если за 1 тик вызвать несколько методов передвижения, реально вызовется только последний (например в коде выше это this.down). Если ни один из методов не был вызван, танк остановится в направлении предыдущего движения.
```js
// Огонь
this.fire();
```
Попытка выстрелить. Огонь ведётся только в направлении движения (башня не крутится). На старте у бота есть 3 снаряда, которые можно расстрелять за три следующих тика. Снаряды восстанавливаются по 1 штуке за 10 тиков до максимального значения в 3 снаряда.
```js
// Ускорение
this.nitro();
```
Резкое ускорение в направлении движения. На старте у бота есть 6 литров закиси азота, которые можно потратить за 6 следующих тиков. Затем закись восстанавливается по 1 литру за 10 тиков, однако вызов метода nitro ставит на паузу это восстановление на 10 тиков. Иными словами, если постоянно дёргать метод, ускорения не будет вообще.
```js
// Противник
this.enemy;
```
Это поле, в котором хранится текущий преследуемый противник. Назначается автоматически. Можно задать вручную через метод
```js
// Преследование противника
this.pursue(enemyId || undefined);
```
Записывает в this.enemy случайного бота, либо переданного в первом аргументе, и начинает его преследовать. По сути, this.pursue - это автоматический вызов одного из четырёх методов передвижения для максимально быстрого выхода на одну линию с противником.
```js
// Убегание
this.escape(enemyId || undefined);
```
Автоматический вызов одного из четырёх методов передвижения для максимального удаления от текущего противника.

У танка есть набор флагов, характеризующих его состояние:
```js
this.health; // Здоровье танка, 0-100.
this.armed; // Параметр готовности стрельбы, 0-30. Выстрел отнимает 10.
this.immortal; // Число тиков до потери бессмертия. При попытке выстрелить бессмертие теряется мгновенно.
this.stamina; // Запас стамины делённый на 10 - количество тиков, в течение которых можно использовать метод nitro. Вызов метода nitro отменяет восстановление стамины на 10 тиков. 0-60.
```

Особый набор флагов находится в объекте
```js
this.powerups == {
    '2gisDamage': 25
};
```
В этом примере у танка есть '2gisDamage', который будет сохраняться следующие 25 тиков (либо до смерти). То есть следующие 25 тиков у снарядов танка будет двойной урон.

### Данные о текущей игре
Все данные о других танках, границах карты, летящих снарядах и поверапах находятся в объекте frame, который имеет следующую структуру:
```js
this.frame = {
    // Живые боты на карте
    players: [{
        id: 123, // Уникальный идентификатор бота
        name: 'overmind', // Никнейм бота
        x: 46, // Положение бота, карта обычно размером 100 на 120
        y: 84,
        width: 5, // Стандартный размер танка
        height: 5,
        health: 100, // Здоровье бота, максимум 100
        immortal: 6, // Количество тиков до потери бессмертия
        // Массив поверапов, в данном тике подцепленных игроком
        powerups: [{
            type: '2gisDamage',
            left: 12 // Количество тиков, после которого поверап кончится
        }],
    }],
    enemy: ... // Случайный бот, являющийся текущей целью
    // Массив летящих снарядов
    shells: [{
        id: '123', // Уникальный идентификатор снаряда
        vector: [0, 1], // Направление полёта снаряда, в данном случае вниз (Y направлен вниз)
        x: 123,
        y: 25,
        shooter: '123' // Идентификатор бота, который выпустил этот снаряд
    }],
    // Массив поверапов, в данном тике находящихся на карте
    powerups: [{
        type: '2gisDamage',
        left: 12
    }],
    map: {
        size: { // Размеры карты
            x: 100,
            y: 120
        }
    }
}
```

### Определение своих методов
Для структурирования кода бота полезно выносить какие-то блоки кода в отдельные функции или методы самого бота. Поскольку код выполняется каждый тик, для экономии ресурсов рекомендуется использовать следующий паттерн:
```js
this.fireForSure = this.fireForSure || function() {
    if (this.locked()) this.fire();
}

this.pursue();
this.fireForSure(); // Будет стрелять только когда противник на линии огня
```
В таком коде метод создастся только раз при первом выполнении.

В прототипе инстанса бота есть методы типа pursue и escape, но они не идеальны и их можно переопределять указанным способом.

### Рандомный бот

Самый простой бот, который передвигается в случайном направлении и постоянно стреляет:
```js
var moves = ['left', 'up', 'right', 'down'];
var method = moves[Math.foor(Math.random() * 4)];

this[method]();
this.fire();
```

## Физика

Все размеры - условные пиксели. Это могут быть сантиметры, километры или парсеки, не важно. Но для простоты будем называть эти единицы пикселями. Главное понимать, что это не экранные, логические, физические или CSS пиксели. Это просто условные единицы длины.

У бота есть скорость. При простом передвижении она равняется 1 пиксель в тик, но при ускорении она повышается. При скорости больше 2 у танка появляется инерция: при повороте его будет заносить. Существует отдача, которая добавляет вектор скорости 1 в сторону, противоположную выстрелу.

Бот может стрелять когда его параметр armed больше 10. Он увеличивается каждый тик и может достигать максимум 30. Каждый выстрел уменьшает его на 10. То есть, на старте или после долгого перерыва можно быстро сделать три выстрела, но потом скорострельност резко снижается до 1 в 10 тиков.

Бот не может выехать за пределы карты. Его перемещение также ограничено другими ботами. Координаты любого бота - это координаты его левого верхнего угла.

После респавна у бота есть бессмертие - параметр immortal имеющий значение 10 и уменьшающийся на 1 каждый тик. Он обнуляется мгновенно при попытке выстрелить.
