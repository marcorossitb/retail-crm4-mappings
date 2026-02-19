# Customers Picklist Mappings (AIDEA 1_0_0_0)

## Obiettivo

Documentare le entità picklist usate dal frontend nell'area Customers.
Queste entità vengono tutte lette da una stessa stored procedure SQL (`tb_picklist_list_g`) e condividono quindi un contratto comune (parametri, struttura risposta, ecc.) che viene descritto in dettaglio nella sezione "Contratto comune" più avanti.

## Contratto comune

- Metodo HTTP: `GET`
- Path params: nessuno (in tutti gli endpoint sotto)
- Body: nessuno (in tutti gli endpoint sotto)
- Stored procedure principale: `tb_picklist_list_g`
- Parametri tecnici SQL sempre inviati da `RetailConnection`: `SystemUserId`, `LineCode`, `RoleName`
- Struttura risposta: `IEntityPaginatedList<T>`

### Mappatura più comune (campi disponibili in `tb_picklist_list_g`):

- `id` -> `PicklistId` oppure `CRMIntCode` (a seconda dell'entità)
- `code` -> `ItemCode` oppure `ItemCodeAbbr` (a seconda dell'entità)
- `name` -> `ItemCodeName`
- `isDefault` -> `IsDefault` oppure `ChildIsDefault` (a seconda dell'entità)
- Campi extra: `ItemValue1`, `SortOrder`, `Extra` (a seconda dell'entità)

Esempio response envelope:

```json
{
  "items": [
    {
      "id": "33387",
      "code": "X",
      "name": "Nome tipo cliente",
      "abbr": "NTC",
      "isDefault": true
    }
  ],
  "pagination": {
    "itemCountGlobal": 1,
    "itemCountAfterFilterAndPagination": 1
  }
}
```

## Entita raggruppate per stored procedure

### `tb_picklist_list_g` (principale)

- `CustomerAccountStatus`
  - Filtro distintivo: `Entity='Account.Status'`
  - Descrizione: picklist stato account cliente.

- `CustomerBusinessRole`
  - Filtro distintivo: `Entity='MS_AccountCustomerType'`
  - Descrizione: picklist ruolo business con mappatura non standard (`id` da `CRMIntCode`).

- `CustomerClass`
  - Filtro distintivo: `Entity='accountclass'`
  - Descrizione: picklist classe cliente.

- `CustomerCluster`
  - Filtro distintivo: `Entity='accountcluster'` oppure `Entity='MS_accountcluster'` se `isMultiselection=true`
  - Descrizione: picklist cluster cliente; con `isMultiselection=true` l'`id` proviene da `CRMIntCode`.

- `CustomerClusterType`
  - Filtro distintivo: `Entity='Tier'`
  - Descrizione: picklist tipo cluster.
  - Specifica: campo extra `sortOrder` (da `SortOrder`).

- `CustomerCompanyType`
  - Filtro distintivo: `Entity='CompanyType'`
  - Descrizione: picklist tipo società cliente (`isDefault` da `IsDefault`).

- `CustomerConditionType`
  - Filtro distintivo: base `Entity = 'ms_conditiontype'`; con `customerId` aggiunge `AND ItemCode IN (...)`
  - Descrizione: picklist tipi condizione; opzionalmente filtrata sulle opzioni associate al cliente.

- `CustomerConditionDeadline`
  - Filtro distintivo: base `Entity = 'ms_conditiondeadline'`; con `customerId` aggiunge `AND ItemCode IN (...)`
  - Descrizione: picklist scadenze condizione; opzionalmente filtrata sulle opzioni associate al cliente.

- `CustomerConnectionType`
  - Filtro distintivo: `Entity='account_r.entity'`
  - Descrizione: picklist tipi connessione con mappatura non standard (`id` da `ItemCode`, `code` da `ItemCodeAbbr`).

- `CustomerDeferredPayment`
  - Filtro distintivo: `ChildEntity='DeferredPayment' AND PicklistId='<paymentModeId>'`
  - Descrizione: picklist dipendente dalla modalita pagamento (`paymentModeId` obbligatorio).

- `CustomerDeliveryBlock`
  - Filtro distintivo: `Entity='deliveryblock'`
  - Descrizione: picklist blocco consegna.
  - Specifica: campo extra `status` (da `ItemValue1`).

- `CustomerDocumentType`
  - Filtro distintivo: base `Entity = 'erp_attachmenttype'`; opzionale `AND ItemCode = '<itemCode>'`
  - Descrizione: picklist tipi documento con filtri opzionali (`itemCode`, `getFilesCount`, `parentId`).
  - Specifica: campo extra `extra` quando `getFilesCount=true`.

- `CustomerExternalDeliveryBlock`
  - Filtro distintivo: `Entity='deliveryblockexternal'`
  - Descrizione: picklist blocco consegna esterno.
  - Specifica: campo extra `color` (da `ItemValue1`).

- `CustomerMaxiGroup`
  - Filtro distintivo: `Entity='AccountGroup'`
  - Descrizione: picklist maxi gruppo cliente.

- `CustomerPOA`
  - Filtro distintivo: `Entity='account_line.segmentation1'`
  - Descrizione: picklist POA/segmentazione.

- `CustomerPaymentMode`
  - Filtro distintivo: `Entity='PaymentMode'`
  - Descrizione: picklist modalita pagamento.

- `CustomerSplitPayment`
  - Filtro distintivo: `Entity='YesNo'`
  - Descrizione: picklist split payment.
- `CustomerStatus`
  - Filtro distintivo: `Entity='Account.ErpStatus'`
  - Descrizione: picklist stato ERP cliente.

- `CustomerType`
  - Filtro distintivo: `ChildEntity='AccountType' AND CRMIntCode in ('33387')`
  - Descrizione: picklist tipo cliente con campi sorgente `Child*` e `id` da `ChildCRMIntCode`.

### `tb_account_list_g`

- `CustomerConditionType`
  - Uso: se arriva `customerId`, legge `MS_ConditionType_ItemCode` per costruire il filtro `ItemCode IN (...)` sulla picklist.

- `CustomerConditionDeadline`
  - Uso: se arriva `customerId`, legge `MS_ConditionDeadline_ItemCode` per costruire il filtro `ItemCode IN (...)` sulla picklist.

### `tb_attachment_list_g`

- `CustomerDocumentType`
  - Uso: se `getFilesCount=true`, esegue una query per ogni tipo documento e valorizza `extra` con il numero file.

## Dettagli implementativi importanti

- `CustomerDocumentTypeRepository`: con `getFilesCount=true` esegue una query aggiuntiva `tb_attachment_list_g` per ogni document type.
- `CustomerConditionTypeRepository` e `CustomerConditionDeadlineRepository`: se arriva `customerId`, fanno prima una query su `tb_account_list_g` per restringere i codici validi.
- `CustomerPaymentModeRepository`: usa `orderby` (minuscolo) nel parametro request verso SQL.

## Riferimenti codice

- Router: `api/src/Presentation/Common/router.ts`
- Presentation handlers: `api/src/Presentation/Customers/getCustomer*.ts`, `api/src/Presentation/Customers/getCustomers*.ts`
- Repositories: `api/src/Infrastructure/AIDEA/1_0_0_0/Customers/*Repository.ts`
- Connessione SQL retail: `api/src/Infrastructure/AIDEA/Common/RetailConnection.ts`
- Entita dominio: `api/src/Domain/Customers/*.ts`
