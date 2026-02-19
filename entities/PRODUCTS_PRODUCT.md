# Products - Product Repository Mapping

## Obiettivo

Riepilogo operativo dell'entita `Product` usata dal frontend per ottenere l'elenco prodotti con eventuale filtro per stato.

## Parametri in ingresso (Presentation)

- Query supportata da endpoint:
  - `statusId?`

## Parametri repository supportati

- `filters.statusId?`
- `filters.productId?`
- `filters.productName?`

## Stored procedure usata

- `tb_product_list_g_standard`

## Parametri principali inviati alla SP

- `filter`: sempre `Entity='erp'`, con filtri opzionali aggiunti:
  - `ProductStatusId='<statusId>'`
  - `ProductId='<productId>'`
  - `ProductCodeName like '%<productName>%'`
- `orderBy='ProductCodeName'`
- `fields='ProductId, ProductCode, ProductCodeName, ParentId, ParentCode, ParentCodeName'`

## Esempio mapping output

```ts
new Product({
  id: external.ProductId,
  code: external.ProductCode,
  name: external.ProductCodeName,
  brand: external.ParentId
    ? new OrderBrand({
        id: external.ParentId,
        code: external.ParentCode,
        name: external.ParentCodeName,
      })
    : undefined,
});
```

## Struttura risposta API

- `IEntityPaginatedList<Product>`:
  - `items[]` con campi `id`, `code`, `name`, `brand?`
  - `pagination.itemCountGlobal`
  - `pagination.itemCountAfterFilterAndPagination`

## Note importanti

- L'endpoint `GET /v1/products` espone solo `statusId`, ma il repository supporta anche `productId` e `productName`.
