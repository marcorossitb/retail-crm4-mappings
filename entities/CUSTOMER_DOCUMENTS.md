# Customers - Customer Documents Mapping

## Obiettivo
Riepilogo operativo dell'entita `CustomerDocument` usata dal frontend.

## Operazioni API coperte
- `GET /v1/customers/:customerId/documents`
- `GET /v1/customers/:customerId/documents/storagelink`
- `POST /v1/customers/:customerId/documents`
- `PUT /v1/customers/:customerId/documents/:documentId`
- `DELETE /v1/customers/:customerId/documents/:documentId`

## Parametri in ingresso (Presentation)
- `GET .../documents`: path `customerId`, query `typeId?`, `fullTextFilter?`, `pageNumber?`, `pageItemCount?`
- `GET .../documents/storagelink`: path `customerId`, query `fileName`, `permission` (`w|r|d`)
- `POST .../documents`: path `customerId`, body `description`, `fileName`, `fileType`, `fileSize`, `documentTypeId?`
- `PUT .../documents/:documentId`: path `customerId`, `documentId`, body `description?`, `loadDate?`, `fileName?`, `fileType?`, `fileSize?`, `documentTypeId?`
- `DELETE .../documents/:documentId`: path `customerId`, `documentId`, body `fileName`

## Stored procedure usate
- Lista documenti: `tb_attachment_list_g`
- Inserimento documento: `tb_attachment_i`
- Aggiornamento documento: `tb_attachment_u`
- Cancellazione documento: `tb_attachment_d`

## Parametri principali inviati alle stored procedure
- `getList` su `tb_attachment_list_g`: `parentId`, `filter` (su `AttachmentTypeId` e/o full text), `fields='ParentId, AttachmentId, AttachmentDescription, FileName, FileSize, LoadDate, app_user'`
- `createSingle` / `createSingleWithUploadLink` su `tb_attachment_i`: `ParentId`, `AttachmentId`, `AttachmentDescription`, `AttachmentTypeId`, `Date`, `LoadDate`, `Entity='retail'`, `FileName`, `FilePath`, `FileSize`, `FileType`
- `updateSingle` / `updateSingleWithUploadLink` su `tb_attachment_u`: `AttachmentId`, `AttachmentDescription`, `AttachmentTypeId`, `LoadDate`, `FileName`, `FilePath`, `FileSize`, `FileType`
- `deleteSingle` / `deleteSingleWithDeleteLink` su `tb_attachment_d`: `AttachmentId`

## Mapping dati in uscita (repository -> entity)
Esempio (estratto semplificato di `mapToEntity` + arricchimento owner):

```ts
const document = new CustomerDocument({
  id: external.AttachmentId,
  parentId: external.ParentId,
  description: external.AttachmentDescription,
  fileName: external.FileName,
  fileSize: external.FileSize,
  loadDate: retailDateToJs(external.LoadDate, undefined),
});

const appUserId = external.app_user.toLowerCase();
const user = await this.externalUserRepository.getSingle(appUserId);

document.owner = {
  id: user.id,
  name: user.fullName,
};
```

In sintesi: i campi DB `Attachment*` e `File*` popolano il `CustomerDocument`, mentre `app_user` viene risolto a nome utente tramite `ExternalUserRepository`.

## Link storage (Azure SAS)
- `getSingleStorageLink`: genera URL temporaneo con permesso `r|w|d` su blob `retail/documents/<PARENT_ID>/<fileName>`
- `createSingleWithUploadLink`: genera `fileUploadUrl` (permesso `w`) e salva `filePath` senza query string
- `updateSingleWithUploadLink`: rigenera `fileUploadUrl` se cambia/allega file
- `deleteSingleWithDeleteLink`: genera `fileDeleteUrl` (permesso `d`) prima della delete DB

## Struttura risposta API
- `GET .../documents`: `IEntityPaginatedList<CustomerDocument>` con `items[]` e `pagination`
- `GET .../documents/storagelink`: `{ link }`
- `POST .../documents`: documento creato (include `id`, `parentId`, eventuale `fileUploadUrl`)
- `PUT .../documents/:documentId`: documento aggiornato (eventuale nuovo `fileUploadUrl`)
- `DELETE .../documents/:documentId`: documento eliminato (ritorna dati con eventuale `fileDeleteUrl`)

## Note importanti
- Nel repository `getList` non applica realmente la paginazione `pageNumber/pageItemCount` ai parametri SQL, anche se il layer presentation li espone.
- Il mapping owner richiede lookup utente esterno (`ExternalUserRepository`) per ottenere il nome completo.
