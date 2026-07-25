# client

Package client defines the public API shared by frag-engine client implementations.

Client is the main entry point. It groups the engine's capabilities into focused services for files, search, access management, object registration, profiles, indexing, and group folders. The package also owns the request, response, actor, and normalized error types shared by those services.
## Table of Contents

- [Implementations](#implementations)
- [Profiles and actors](#profiles-and-actors)
- [Errors](#errors)
- [Constants](#constants)
- [Functions](#functions)
  - [TestProfileModuleRoundTrip](#testprofilemoduleroundtrip)
  - [WithActor](#withactor)
- [Types](#types)
  - [AccessService](#accessservice)
  - [Account](#account)
  - [Actor](#actor)
  - [Client](#client)
  - [Error](#error)
    - [Error.Error](#errorerror)
    - [Error.Unwrap](#errorunwrap)
  - [ErrorCode](#errorcode)
  - [ExtractionMode](#extractionmode)
  - [FileFieldExpanded](#filefieldexpanded)
  - [FileObj](#fileobj)
  - [FileObjExpand](#fileobjexpand)
  - [FileObjectLink](#fileobjectlink)
  - [FileObjectLinkQuery](#fileobjectlinkquery)
  - [FileObjectTagUpdate](#fileobjecttagupdate)
  - [FilePathSuggestion](#filepathsuggestion)
  - [FilePathSuggestionHint](#filepathsuggestionhint)
  - [FilePathSuggestionRequest](#filepathsuggestionrequest)
  - [FilePathSuggestionService](#filepathsuggestionservice)
  - [FileService](#fileservice)
  - [FileTags](#filetags)
  - [FolderGrant](#foldergrant)
  - [GroupFolderService](#groupfolderservice)
  - [IndexFolderRequest](#indexfolderrequest)
  - [IndexFolderResponse](#indexfolderresponse)
  - [IndexerService](#indexerservice)
  - [InstanceGrant](#instancegrant)
  - [Object](#object)
  - [ObjectRef](#objectref)
  - [ObjectSearchResult](#objectsearchresult)
  - [ObjectType](#objecttype)
  - [Permission](#permission)
  - [PrincipalReference](#principalreference)
  - [Profile](#profile)
  - [ProfileService](#profileservice)
  - [ProvisioningProfile](#provisioningprofile)
  - [RegistryService](#registryservice)
  - [SearchQuery](#searchquery)
  - [SearchResponse](#searchresponse)
  - [SearchResult](#searchresult)
  - [SearchService](#searchservice)
  - [SearchSuggestion](#searchsuggestion)
  - [Tag](#tag)
  - [UserGrant](#usergrant)
  - [UserNote](#usernote)
  - [Visibility](#visibility)

## Implementations

The client/frag package provides the production implementation backed by Nextcloud and Elasticsearch. The client/fake package provides an in-memory implementation for tests and local development. Both satisfy Client, so callers can depend on this package without depending on a concrete backend.

## Profiles and actors

Client.WithProfile returns a view scoped to a named configuration profile. Calls made through the original client continue to use its original profile.

WithActor attaches the calling user's identity and known permissions to a context. Search and routing services can use that information to restrict results without changing every method signature.

## Errors

Normalized service failures use \*Error. Callers can inspect them with errors.As and branch on Error.Code instead of matching backend-specific messages. Error.Unwrap preserves an underlying cause when one is available.

A typical production setup looks like:

	engine := fragclient.NewClient(nextcloudClient, elasticsearchClient)
	scoped := engine.WithProfile("legal-review")
	ctx := client.WithActor(context.Background(), client.Actor{
		ExternalID: "user-42",
		Groups:     []string{"legal"},
	})

	result, err := scoped.Search().Search(ctx, client.SearchQuery{
		Query: "quarterly report",
	})
## Constants

```go
const (
	FilePathSuggestionSourceVector   = "vector"
	FilePathSuggestionSourceFilename = "filename"
	FilePathSuggestionSourceHistory  = "history"
	FilePathSuggestionSourceHybrid   = "hybrid"
	FilePathSuggestionSourceLLM      = "llm"
	FilePathSuggestionSourceFallback = "fallback"
)
```

```go
const (
	SearchResultKindFile   = elasticsearchapi.SearchResultKindFile
	SearchResultKindObject = elasticsearchapi.SearchResultKindObject
)
```

```go
const (
	SearchSuggestionKindFile   = "file"
	SearchSuggestionKindFolder = "folder"
	SearchSuggestionKindObject = SearchResultKindObject
)
```

```go
const DefaultProfileID = profilepkg.DefaultID
```

## Functions

### TestProfileModuleRoundTrip

```go
func TestProfileModuleRoundTrip
```

### WithActor

```go
func WithActor context.Context
```

WithActor stores the caller description into the context so downstream services can read it without adding extra parameters everywhere.

## Types

### AccessService

```go
type AccessService interface {
	CreateUser(ctx context.Context, profile ProvisioningProfile) (*Account, error)
	DeleteUser(ctx context.Context, ref PrincipalReference) error
	GrantInstanceRights(ctx context.Context, grant InstanceGrant) error
	RevokeInstanceRights(ctx context.Context, grant InstanceGrant) error
	GrantUserRights(ctx context.Context, grant UserGrant) error
	RevokeUserRights(ctx context.Context, grant UserGrant) error
	GrantFolderRights(ctx context.Context, grant FolderGrant) error
	RevokeFolderRights(ctx context.Context, grant FolderGrant) error
	LookupAccount(ctx context.Context, ref PrincipalReference) (*Account, error)
}
```

AccessService keeps Nextcloud user management delegated to frag/site code by exposing creation, deletion, and rights-management helpers.

### Account

```go
type Account struct {
	ExternalID  string
	NextcloudID string
	Username    string
	Email       string
}
```

Account represents the Nextcloud account that mirrors a site user.

### Actor

```go
type Actor struct {
	ExternalID string
	Roles      []string
	Groups     []string
	RightTags  []uint16
}
```

Actor represents the calling site user and the rights it already knows about (e.g., Keycloak roles). Frag uses this to filter search results without asking Nextcloud since rights are synced through AccessService.

### Client

```go
type Client interface {
	WithProfile(name string) Client

	Files() FileService
	Search() SearchService
	Access() AccessService
	Registry() RegistryService
	Profiles() ProfileService
	Indexer() IndexerService
	GroupFolders() GroupFolderService
}
```

Client is the main entry point for frag-engine capabilities. WithProfile returns a profile-scoped view; the other methods expose focused services for each capability.

### Error

```go
type Error struct {
	Code    ErrorCode
	Status  int
	Op      string
	Target  string
	Message string
	Err     error
}
```

Error normalizes failures returned by frag so callers are not exposed to Nextcloud/Elasticsearch specific wording.

#### Methods

##### Error

```go
func (e *Error) Error string
```

##### Unwrap

```go
func (e *Error) Unwrap error
```

### ErrorCode

```go
type ErrorCode string
```

ErrorCode identifies a normalized category of client failure.

### ExtractionMode

```go
type ExtractionMode string
```

ExtractionMode controls how the resolver treats candidates for an object type after extraction.

### FileFieldExpanded

```go
type FileFieldExpanded struct {
	Tags []Tag `json:"tags_ids,omitempty"`
}
```

FileFieldExpanded captures additional expanded fields for a file.

### FileObj

```go
type FileObj struct {
	Displayname     string    `json:"displayname"`
	Path            string    `json:"path"`
	AppID           string    `json:"app_id,omitempty"`
	TenantID        string    `json:"tenant_id,omitempty"`
	Description     string    `json:"description"`
	Mimetype        string    `json:"mimetype"`
	LastModified    time.Time `json:"lastmodified"`
	MountType       string    `json:"mountype"`
	Content         string    `json:"content"`
	FileID          int32     `json:"fileid"`
	Tags            FileTags  `json:"tags"`
	TaggerLastError string    `json:"tagger_last_error,omitempty"`
}
```

FileObj mirrors metadata for a file from frag-engine.

### FileObjExpand

```go
type FileObjExpand struct {
	File     *FileObj           `json:"file"`
	Expanded *FileFieldExpanded `json:"expanded,omitempty"`
	UserNote *UserNote          `json:"user_note,omitempty"`
}
```

FileObjExpand is the payload returned by expanding file metadata.

### FileObjectLink

```go
type FileObjectLink = elas.FileObjectLinkDocument
```

FileObjectLink records a generic relationship between a file and an app-defined object.

### FileObjectLinkQuery

```go
type FileObjectLinkQuery = elas.FileObjectLinkQuery
```

FileObjectLinkQuery filters generic file-object links created by the engine or synced by an app.

### FileObjectTagUpdate

```go
type FileObjectTagUpdate = elas.FileObjectTagUpdate
```

FileObjectTagUpdate replaces the tag entry of one object type on a file.

### FilePathSuggestion

```go
type FilePathSuggestion struct {
	Path     string  `json:"path"`
	Source   string  `json:"source"`
	Score    float64 `json:"score,omitempty"`
	Existing bool    `json:"existing"`
}
```

FilePathSuggestion is one candidate folder path for the incoming document.

### FilePathSuggestionHint

```go
type FilePathSuggestionHint struct {
	Path   string  `json:"path"`
	Weight float64 `json:"weight,omitempty"`
	Source string  `json:"source,omitempty"`
}
```

FilePathSuggestionHint supplies a caller-owned routing signal, such as a recently confirmed destination for a similar document. Path is still checked against the verified directory tree before it can be returned.

### FilePathSuggestionRequest

```go
type FilePathSuggestionRequest struct {
	FileID            string                   `json:"file_id"`
	AppID             string                   `json:"app_id,omitempty"`
	TenantID          string                   `json:"tenant_id,omitempty"`
	BaseFolder        string                   `json:"base_folder,omitempty"`
	RootPath          string                   `json:"root_path,omitempty"`
	ExcludePaths      []string                 `json:"exclude_paths,omitempty"`
	Limit             int                      `json:"limit,omitempty"`
	NeighborLimit     int                      `json:"neighbor_limit,omitempty"`
	SearchPerVector   int                      `json:"search_per_vector,omitempty"`
	MaxSourceVectors  int                      `json:"max_source_vectors,omitempty"`
	MaxDirectoryDepth int                      `json:"max_directory_depth,omitempty"`
	MaxExistingPaths  int                      `json:"max_existing_paths,omitempty"`
	Hints             []FilePathSuggestionHint `json:"hints,omitempty"`
}
```

FilePathSuggestionRequest asks frag to recommend existing folders for an indexed file, using the vectors already stored for FileID as the semantic source.

### FilePathSuggestionService

```go
type FilePathSuggestionService interface {
	SuggestFilePaths(ctx context.Context, request FilePathSuggestionRequest) ([]FilePathSuggestion, error)
}
```

FilePathSuggestionService suggests suitable storage paths for files.

### FileService

```go
type FileService interface {
	Get(ctx context.Context, path string) (*file.File, error)
	Upload(ctx context.Context, doc *documents.File) error
	GetMetadata(ctx context.Context, doc documents.Document) (documents.Document, error)
	GetFilesMetadatas(ctx context.Context, docs []documents.Document) ([]documents.Document, error)
	Move(ctx context.Context, src, dst string) error
	Share(ctx context.Context, path string, visibility Visibility) error
	CreateFolder(ctx context.Context, path string, visibility Visibility) error
	LookupPath(ctx context.Context, path string) (*documents.File, error)
	DeleteFolder(ctx context.Context, path string)
	ChangeFolderVisibility(ctx context.Context, path string, visibility Visibility) error
}
```

FileService manages file content, metadata, paths, and visibility.

### FileTags

```go
type FileTags []Tag
```

FileTags groups tags applied to a file.

### FolderGrant

```go
type FolderGrant struct {
	FolderPath  string
	TargetID    string
	Permissions []Permission
}
```

FolderGrant specializes the permissions on a given folder.

### GroupFolderService

```go
type GroupFolderService = nextcloudapi.GroupFolderApi
```

GroupFolderService provides access to Nextcloud group folder management This is an alias for the existing GroupFolderApi interface

### IndexFolderRequest

```go
type IndexFolderRequest = indexerClients.ForceIndexFolderRequest
```

### IndexFolderResponse

```go
type IndexFolderResponse = indexerClients.ForceIndexFolderResponse
```

### IndexerService

```go
type IndexerService interface {
	ForceIndexFolder(ctx context.Context, folder string) (*IndexFolderResponse, error)
}
```

### InstanceGrant

```go
type InstanceGrant struct {
	UserID      string
	Permissions []Permission
	QuotaBytes  int64
}
```

InstanceGrant configures access that applies at the Nextcloud instance level (e.g., enable/disable apps or storage quotas).

### Object

```go
type Object struct {
	AppID          string         `json:"app_id"`
	TenantID       string         `json:"tenant_id"`
	Type           string         `json:"type"`
	ID             string         `json:"object_id"`
	DisplayName    string         `json:"display_name"`
	NormalizedName string         `json:"normalized_name,omitempty"`
	Aliases        []string       `json:"aliases,omitempty"`
	Attributes     map[string]any `json:"attributes,omitempty"`
}
```

Object represents one app-owned object made available to the generic extraction registry.

### ObjectRef

```go
type ObjectRef struct {
	AppID    string `json:"app_id"`
	TenantID string `json:"tenant_id"`
	Type     string `json:"type"`
	ID       string `json:"object_id"`
}
```

ObjectRef identifies an app-owned object without requiring the full object payload.

### ObjectSearchResult

```go
type ObjectSearchResult = elasticsearchapi.SearchCommandObjectResult
```

### ObjectType

```go
type ObjectType struct {
	AppID       string         `json:"app_id"`
	TenantID    string         `json:"tenant_id"`
	ProfileID   string         `json:"profile_id"`
	Type        string         `json:"type"`
	Label       string         `json:"label"`
	Description string         `json:"description,omitempty"`
	Mode        ExtractionMode `json:"mode"`
	Instruction string         `json:"instruction,omitempty"`
	Schema      map[string]any `json:"schema,omitempty"`
}
```

ObjectType defines a generic kind of object that the engine can extract or link without knowing the app-specific business meaning.

### Permission

```go
type Permission = string
```

Permission is a coarse-grained flag describing the action granted.

### PrincipalReference

```go
type PrincipalReference struct {
	ExternalID  string
	NextcloudID string
	Username    string
	Email       string
}
```

PrincipalReference lets callers look up a subject using any combination of identifiers they already know.

### Profile

```go
type Profile = profilepkg.Profile
```

### ProfileService

```go
type ProfileService interface {
	Upsert(profile *Profile) error
	Get(name ...string) (*Profile, error)
	Module(module string, name ...string) (json.RawMessage, error)
}
```

ProfileService stores modular app profiles. Built-in packages can claim one module key, such as "tagger", while the profile identity stays generic.

### ProvisioningProfile

```go
type ProvisioningProfile struct {
	ExternalID  string
	Username    string
	Email       string
	DisplayName string
	Locale      string
	Roles       []string
	Groups      []string
	Attributes  map[string][]string
}
```

ProvisioningProfile is the payload used to create or update a Nextcloud account from the site.

### RegistryService

```go
type RegistryService interface {
	UpsertObjectType(ctx context.Context, typ ObjectType) error
	UpsertObject(ctx context.Context, obj Object) error
	DeleteObject(ctx context.Context, ref ObjectRef) error
	LinkFileObject(ctx context.Context, link FileObjectLink) error
	DeleteFileObjectLinks(ctx context.Context, query FileObjectLinkQuery) error
	ReplaceFileObjectTag(ctx context.Context, update FileObjectTagUpdate) error
	ListFileObjectLinks(ctx context.Context, query FileObjectLinkQuery) ([]FileObjectLink, error)
}
```

RegistryService syncs app-defined object metadata into frag-engine.

### SearchQuery

```go
type SearchQuery = elasticsearchapi.SearchCommandQuery
```

### SearchResponse

```go
type SearchResponse = elasticsearchapi.SearchCommandResponse
```

### SearchResult

```go
type SearchResult = elasticsearchapi.SearchCommandResult
```

### SearchService

```go
type SearchService interface {
	Index(ctx context.Context, path string, doc *file.File, visibility Visibility) error
	Query(ctx context.Context, query string, k int) ([]elasticsearchapi.FileSearchResult, error)
	Autocomplete(ctx context.Context, query SearchQuery) ([]SearchSuggestion, error)
	Search(ctx context.Context, query SearchQuery) (*SearchResponse, error)
	FilePathSuggestionService
}
```

SearchService indexes files and exposes the available search workflows.

### SearchSuggestion

```go
type SearchSuggestion struct {
	ID             string              `json:"id"`
	Kind           string              `json:"kind"`
	Label          string              `json:"label"`
	Path           string              `json:"path,omitempty"`
	DisplayPath    string              `json:"display_path,omitempty"`
	Description    string              `json:"description,omitempty"`
	Extension      string              `json:"extension,omitempty"`
	Mimetype       string              `json:"mimetype,omitempty"`
	ObjectType     string              `json:"object_type,omitempty"`
	NormalizedName string              `json:"normalized_name,omitempty"`
	Aliases        []string            `json:"aliases,omitempty"`
	Attributes     map[string]any      `json:"attributes,omitempty"`
	AppID          string              `json:"app_id,omitempty"`
	TenantID       string              `json:"tenant_id,omitempty"`
	Score          float64             `json:"score,omitempty"`
	Highlights     map[string][]string `json:"highlights,omitempty"`
}
```

SearchSuggestion is a compact result row intended for autocomplete UIs.

### Tag

```go
type Tag struct {
	ID            string `json:"-"`
	Type          string `json:"tag_type"`
	DisplayName   string `json:"display_name"`
	NormalizeName string `json:"normalize_name"`
	SynonymSet    string `json:"synonym_set"`
	SynonymRuleID string `json:"synonym_rule_id"`
	CreatedAt     string `json:"created_at"`
}
```

Tag describes a label returned by frag-engine.

### UserGrant

```go
type UserGrant struct {
	OwnerUserID  string
	TargetUserID string
	Permissions  []Permission
}
```

UserGrant describes rights that one user has on another user's space.

### UserNote

```go
type UserNote struct {
	UserID int64  `json:"user_id"`
	Text   string `json:"text"`
}
```

UserNote captures personalized notes attached to a file.

### Visibility

```go
type Visibility struct {
	Users     []string
	Groups    []string
	CompanyID string
	Public    bool
}
```

Visibility represents how a file should be shared inside Nextcloud. Users and Groups expect identifiers managed by the site so frag can mirror permissions when the document is indexed and searched.

