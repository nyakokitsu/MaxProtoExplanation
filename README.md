# MaxProtoExplanation
> Объяснение принципа работы мобильного протокола Max api.oneme.ru

## Шаг 0. Введение
Вероятно все из вас, кто читает этот текст, ознакомлены с работой макса через вебсокет, поэтому мы будем обьяснять отталкиваясь от того что вы в курсе. Если нет - откройте web.max.ru и посмотрите запросы вебсокета.

## Шаг 1. Обьяснение
Пожалуй начать стоит с того, что использовалось при анализе. инструментарий таков: iPhone X(palera1n) с отключенным пиннингом через SSL Kill Switch 2; proxy-сервер Charles; hex-редактор ImHex. Все инструменты бесплатные, пользуйтесь, ковыряйтесь.

Сняв пакеты через Charles мы получаем на руки следующую структуру:

[Заголовок]{Пейлоад}
```
[0b0100070011020000d8]{f3a785a5746f6b656ed988416e5f537836485139484469463436473161617078494932456c4577624237396679374c3339545f3753436b595574646b4d424f487a7a355450717a50446d4b62544371305a76633259315f786e67464641794d506d504f30394346626758676769526f686754394e52687a4e6c6555686838733666513937523450714e4c304f7347584d664d6caa636f64654c656e67746806b2726571756573744d61784475726174696f6ed20000ea60b01800f405436f756e744c6566740ab1616c74416374696f6e290050d20000ea60}
```

Рассмотрим составные заголовка и сопоставим со значениями вебсокетами

0b|0100|07|0011|020000d8
1. ver - версия протокола, принимает значения 10/11.
2. cmd - в теории должен означать тип запроса(от клиента/сервера - 0/1), но тут он очень очень шалит
3. seq - секвенсор, означает какой по счету запрос.
4. opcode - старый добрый опкод, ну кто его не знает
5. payload_data - длина запакованного пейлоада и тип сжатия, значение уникальное для мобильного прото, понадобится нам позже.

Для кодировки payload используется довольно известный messagepack, который вкунтахты используют уже десятилетие, так что это не самая великая тайна.

## Шаг 2: Пердолинг

Давайте попробуем распаковать самый простой пример.
```
0a00000700110000003083a570686f6e65ac2b3739393939393939393939a474797065aa53544152545f41555448a86c616e6775616765a27275
```
Поочередно распакуем заголовок и пейлоад с помощью библиотеки msgpack для python.

```
import msgpack

def unpack_packet(data: bytes):
    ver = int.from_bytes(data[0:1], "big")
    cmd = int.from_bytes(data[1:3], "big")
    seq = int.from_bytes(data[3:4], "big")
    opcode = int.from_bytes(data[4:6], "big")
    payload_length = int.from_bytes(data[6:10], "big")

    payload_bytes = data[10 : 10 + payload_length]
    payload = msgpack.unpackb(payload_bytes, raw=False)

    return {
        "ver": ver,
        "cmd": cmd,
        "seq": seq,
        "opcode": opcode,
        "payload": payload,
    }
```
Результат:
```
{"ver":10,"cmd":0,"seq":7,"opcode":17,
    "payload":{"phone":"+79999999999","type":"START_AUTH","language":"ru"}
```

# Шаг 3: Мега пердолинг
Зная принцип распаковки маленьких пейлоадов, перейдем к большим.
Для упаковки больших пейлоадов используется сжатие через lz4.
Таким образом допишем нашу функцию до нужного вида.

```
import lz4.block
import msgpack

def unpack_packet(data: bytes):
    ver = int.from_bytes(data[0:1], 'big')
    cmd = int.from_bytes(data[1:3], 'big')
    seq = int.from_bytes(data[3:4], 'big')
    opcode = int.from_bytes(data[4:6], 'big')
    packed_len = int.from_bytes(data[6:10], 'big', signed=False)
    comp_flag = packed_len >> 24
    payload_length = packed_len & 0xFFFFFF
    payload_bytes = data[10:10 + payload_length]
    if comp_flag != 0:
        compressed_data = payload_bytes
        try:
            payload_bytes = lz4.block.decompress(compressed_data, uncompressed_size=255)
        except lz4.block.LZ4BlockError:
            return None
    payload = msgpack.unpackb(payload_bytes, raw=False)
    return {
        "ver": ver,
        "cmd": cmd,
        "seq": seq,
        "opcode": opcode,
        "payload": payload
    }
```


# Заключение
Текст писал Дмитрий Уткин, авторы библиотек, ссылайтесь пожалуйста на материал. Спасибо.
