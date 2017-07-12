---
title: Статьи 1С Битрикс | Умный фильтр, бизнес-логика сайта и перестроение фасетного индекса
keywords: sample homepage
sidebar: articles_sidebar
folder: articles
permalink: articles_bx-smart-filter-faset.html
summary: false
toc: true
---

## Умный фильтр, бизнес-логика сайта и перестроение фасетного индекса

### Задача

[Бизнес-логика](https://ru.wikipedia.org/wiki/%D0%91%D0%B8%D0%B7%D0%BD%D0%B5%D1%81-%D0%BB%D0%BE%D0%B3%D0%B8%D0%BA%D0%B0) не всегда логична 
и иногда не реализуема в рамках инструментария, предоставляемого продуктом, а порой заставляет искать довольно странные обходные пути.

Требуется вывести каталог товаров с установленными параметрами фильтрации:
  * Цена товара должна быть больше 0,
  * Показывать только товары у которых есть изображения,
  * Свойство "Показывать на сайте", которое приходит из 1С должно быть в значении "Да"
  
Так же на странице со списком элементов требуется разместить компонент умного фильтра.

### Решение

На первый взгляд все кажется привычно и просто: размещаем компоненты, передаем нужные ограничения в глобальную переменную, 
которая передается в компонент в качестве фильтра "FILTER_NAME" и все должно заработать.

Проблема с которой мы сталкнемся при таком подходе - **в умном фильтре отображается больше характеристик товара, чем есть товаров 
в выводе каталога**. Связано это с тем, что компонент умного фильтра использует [фасетный индекс](https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=42&LESSON_ID=5364)
для [обеспечения своей быстрой работы](https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=43&LESSON_ID=6923).

Для решения этой проблемы вспомним о том, что **индекс строится только по активным товарам**, поэтому нам нужно:
  * **1)** Деактивировать товары, которые не попадают под условия бизнес-лигики отображения товаров на сайте.
  * **2)** Перестроить фасетный индекс с учетом изменений активности товаров
  * **3)** Повесить обработчик события "окончания синхронизации с 1С", выполняющее пункты 1 и 2.
  
#### Деактивация товаров

Битрикс не предоставляет ф-ии для группового изменения элементов инфоблока. Есть метод [CIBlockElement::Update](https://dev.1c-bitrix.ru/api_help/iblock/classes/ciblockelement/update.php), который может обновлять данные элементов по ID, для нашей задачи он не подходит, т.к. нам нужно деактивировать несколько тысяч товаров, а при работе этого метода дополнительно вызываются стандартные события Битрикса OnStartIBlockElementUpdate, OnAfterIBlockElementUpdate, что так же замедлит процесс деактивации.

Поэтому мы деактивируем товары прямым запросом к базе данных, используя возможности "нового" ядра D7

```php
/**
 * Деактивация элементов инфоблок в соответствии с основным фильтром товаров
 * @param int $iblockId
 * @param array $arFilter
 * @return int
 */
function actualizeProducts($iblockId, $arFilter) {
    $arFilterFinal = [
        'IBLOCK_ID' => $iblockId,
        $arFilter,
    ];
    
    // получим список элементов для деактивации
    $dbRes = CIBlockElement::GetList([], $arFilterFinal, false, false, ['ID']);

    $arNotValidIds = [];
    while($arItem = $dbRes->getNext()) {
        $arNotValidIds[] = $arItem['ID'];
    }

    $res = 0;
    if(count($arNotValidIds) > 0) {
        $strNotValidIds = implode(',', $arNotValidIds);
        // формируем запрос на обновления флага активности товаров
        $sql = "UPDATE " . \Bitrix\Iblock\ElementTable::getTableName() . " SET ACTIVE = 'N' WHERE ID IN (" . $strNotValidIds . ")";

        $connection = \Bitrix\Main\Application::getConnection();
        $connection->queryExecute($sql);
        // запускаем переиндексацию фасета, это наша кастомная ф-я
        $res = reindexCatalogFaset($iblockId);
    }

    return $res;
}
```

#### Пересчет фасетного индекса

```php
/**
 * @param $iblockId
 * @return int
 */
public static function reindexCatalogFaset($iblockId) {

    $max_execution_time = 20;

    // Пересоздание фасетного индекса
    // Удалим имеющийся индекс
    Bitrix\Iblock\PropertyIndex\Manager::dropIfExists($iblockId);
    Bitrix\Iblock\PropertyIndex\Manager::markAsInvalid($iblockId);
    
    // Создадим новый индекс
    $index = Bitrix\Iblock\PropertyIndex\Manager::createIndexer($iblockId);
    $index->startIndex();
    $NS = 0;

    do {
        $res = $index->continueIndex($max_execution_time);
        $NS += $res;
    } while($res > 0);

    $index->endIndex();

    // чистим кэши
    \CBitrixComponent::clearComponentCache("bitrix:catalog.smart.filter");
    \CIBlock::clearIblockTagCache($iblockId);

    return $NS;
}
```