# Экзотические ячейки

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

Каждая ячейка имеет свой собственный тип, закодированный целым числом от -1 до 255.
Ячейка с типом -1 является `ordinary` ячейкой, а все остальные ячейки называются `exotic` или `special`.
Тип экзотической ячейки хранится в виде первых восьми бит ее данных. Если экзотическая ячейка имеет менее восьми бит данных, она недействительна.
В настоящее время существует 4 типа экзотических ячеек:

```json
{
  Prunned Branch: 1,
  Library Reference: 2,
  Merkle Proof: 3,
  Merkle Update: 4
}
```

### Обрезанная ветвь

Обрезанные ветви - это ячейки, которые представляют удаленные поддеревья ячеек.

Они могут иметь уровень `1 <= l <= 3` и содержать ровно `8 + 8 + 256 * l + 16 * l` бит.

Первый байт всегда `01` — тип ячейки. Второй — маска уровня обрезанной ветви. Затем идут `l * 32` байта хэшей удаленных поддеревьев, а затем `l * 2` байта глубины удаленных поддеревьев.

Уровень `l` ячейки обрезанной ветви можно назвать ее индексом Де Брейна, поскольку он определяет внешнее доказательство Меркла или обновление Меркла, во время построения которого ветвь была обрезана.

Более высокие хеши обрезанных ветвей хранятся в их данных и могут быть получены следующим образом:

```cpp
Hash_i = CellData[2 + (i * 32) : 2 + ((i + 1) * 32)]
```

### Ссылка на библиотеку

Ячейки ссылок на библиотеку используются для использования библиотек в смарт-контрактах.

Они всегда имеют уровень 0 и содержат `8 + 256` бит.

Первый байт всегда `02` - тип ячейки. Следующие 32 байта - [хеш представления](/v3/documentation/data-formats/tlb/cell-boc#standard-cell-representation-hash-calculation) ячейки библиотеки, на которую делается ссылка.

### Доказательство Меркла

Ячейки доказательства Меркла используются для проверки того, что часть данных дерева ячейки принадлежит полному дереву. Такая конструкция позволяет верификатору не хранить все содержимое дерева, при этом сохраняя возможность проверки содержимого по корневому хешу.

Доказательство Меркла имеет ровно одну ссылку, а ее уровень `0 <= l <= 3` должен быть `max(Lvl(ref) - 1, 0)`. Эти ячейки содержат ровно `8 + 256 + 16 = 280` бит.

Первый байт всегда `03` - тип ячейки. Следующие 32 байта - `Hash_1(ref)` (или `ReprHash(ref)`, если уровень ссылки равен 0). Следующие 2 байта - глубина удаленного поддерева, которое было заменено ссылкой.

Более высокие хеши `Hash_i` ячейки доказательства Меркла вычисляются аналогично более высоким хешам обычной ячейки, но вместо `Hash_i(ref)` используется `Hash_i+1(ref)`.

### Обновление Меркла

Ячейки обновления Меркла всегда имеют 2 ссылки и ведут себя как доказательство Меркла для обоих из них.

Уровень обновления Меркла `0 <= l <= 3` равен `max(Lvl(ref1) − 1, Lvl(ref2) − 1, 0)`. Они содержат ровно `8 + 256 + 256 + 16 + 16 = 552` бит.

Первый байт всегда `04` - тип ячейки. Следующие 64 байта - `Hash_1(ref1)` и `Hash_2(ref2)` - называются старым хешем и новым хешем. Затем идут 4 байта с фактической глубиной удаленного старого поддерева и удаленного нового поддерева.

## Простой пример проверки доказательства

Предположим, что есть ячейка `c`:

```json
24[000078] -> {
	32[0000000F] -> {
		1[80] -> {
			32[0000000E]
		},
		1[00] -> {
			32[0000000C]
		}
	},
	16[000B] -> {
		4[80] -> {
			267[800DEB78CF30DC0C8612C3B3BE0086724D499B25CB2FBBB154C086C8B58417A2F040],
			512[00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000064]
		}
	}
}
```

Но мы знаем только ее хэш `44efd0fdfffa8f152339a0191de1e1c5901fdcfe13798af443640af99616b977`, и мы хотим доказать, что ячейка `a` `267[800DEB78CF30DC0C8612C3B3BE0086724D499B25CB2FBBB154C086C8B58417A2F040]` на самом деле является частью `c`, не получая при этом весь `c`.
Поэтому мы просим доказывающего создать доказательство Меркла, заменив все ветви, которые нам не интересны, на ячейки обрезанных ветвей.

Первый потомок `c`, из которого нет способа добраться до `a`, это `ref1`:

```json
32[0000000F] -> {
	1[80] -> {
		32[0000000E]
	},
	1[00] -> {
		32[0000000C]
	}
}
```

Поэтому доказывающий вычисляет свой хэш (`ec7c1379618703592804d3a33f7e120cebe946fa78a6775f6ee2e28d80ddb7dc`), создает сокращенную ветвь `288[0101EC7C1379618703592804D3A33F7E120CEBE946FA78A6775F6EE2E28D80DDB7DC0002]` и заменяет `ref1` этой сокращенной ветвью.

Второй - `512[0000000...00000000064]`, поэтому проверяющий создает сокращенную ветвь, чтобы заменить и эту ячейку:

```json
24[000078] -> {
	288[0101EC7C1379618703592804D3A33F7E120CEBE946FA78A6775F6EE2E28D80DDB7DC0002],
	16[000B] -> {
		4[80] -> {
			267[800DEB78CF30DC0C8612C3B3BE0086724D499B25CB2FBBB154C086C8B58417A2F040],
			288[0101A458B8C0DC516A9B137D99B701BB60FE25F41F5ACFF2A54A2CA4936688880E640000]
		}
	}
}
```

Результат доказательства Меркла, который проверяющий отправляет верификатору (в данном примере нам), выглядит следующим образом:

```json
280[0344EFD0FDFFFA8F152339A0191DE1E1C5901FDCFE13798AF443640AF99616B9770003] -> {
	24[000078] -> {
		288[0101EC7C1379618703592804D3A33F7E120CEBE946FA78A6775F6EE2E28D80DDB7DC0002],
		16[000B] -> {
			4[80] -> {
				267[800DEB78CF30DC0C8612C3B3BE0086724D499B25CB2FBBB154C086C8B58417A2F040],
				288[0101A458B8C0DC516A9B137D99B701BB60FE25F41F5ACFF2A54A2CA4936688880E640000]
			}
		}
	}
}
```

Когда мы (проверяющий) получаем ячейку Proof, мы убеждаемся, что ее данные содержат хэш `c`, а затем вычисляем `Hash_1` из единственной ссылки Proof: `44efd0fdfffa8f152339a0191de1e1c5901fdcfe13798af443640af99616b977`, и сравниваем его с хешем `c`.

Теперь, когда мы проверили, что хеши совпадают, нам нужно углубиться в ячейку и убедиться, что там есть ячейка `a` (которая нас интересовала).

Такие доказательства многократно снижают вычислительную нагрузку и объем данных, которые необходимо отправить или сохранить в верификаторе.

## См. также

- [Примеры проверки расширенных доказательств](/v3/documentation/data-formats/tlb/proofs)
