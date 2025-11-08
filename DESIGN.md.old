# Backup Tool (bt) - Design Document

## 1. Goals and Non-Goals

### Goals
- Provide a robust personal backup solution for files across multiple computers
- Enable backing up user-selected directories to remote storage
- Support file versioning with ability to view history and restore specific versions
- Use content-addressable storage for automatic deduplication across hosts
- Design with future encryption support in mind
- Maintain simplicity while ensuring reliability for real-world personal use

### Non-Goals
- This is not a shared filesystem or collaboration tool
- Not designed for team or organizational use (single user per host)
- Not implementing incremental or differential backups initially (full backups only)
- Not providing real-time synchronization across hosts
- Not a version control system (no branching, merging, etc.)

## 2. External Dependencies

### Storage (Vault)
The system depends on a storage backend abstraction called a "Vault" which must provide:

**Required Capabilities:**
- **Content Storage**: Ability to store and retrieve content-addressed objects (identified by content checksum)
  - May be cold storage (e.g., S3 Glacier)
  - Must support idempotent writes (duplicate checksums are safely ignored)
- **Metadata Storage**: Ability to store and retrieve metadata files (SQLite databases, JSON files, etc.)
  - Must be identified by host ID
  - Should support versioning or ability to keep multiple versions

**Example Implementations:**
- Local filesystem (for testing)
- AWS S3 or S3-compatible storage (primary use case)
- Other cloud object storage (Azure Blob, Google Cloud Storage, Backblaze B2)

### Host System Requirements
The backup tool requires the following from the host system:
- File system access with ability to read file metadata (permissions, timestamps, etc.)
- Ability to calculate file checksums (SHA-256 or similar)
- SQLite support for local metadata management
- File watcher capabilities (for future daemon implementation)

## 3. Risks and Open Questions

### Known Risks
1. **Large File Handling**: Strategy for handling very large files (e.g., 50GB+ files) is not yet defined
   - Should these be chunked or stored as single objects?
   - Vault implementation detail that needs consideration
   - Impact on memory usage and upload reliability

2. **Metadata Vault Synchronization**: Current design uploads entire SQLite DB to vault after each backup
   - Works for personal use but may become inefficient over time
   - Consider incremental sync strategies in future iterations

3. **Metadata Versioning**: Strategy for keeping multiple versions of metadata in vault is implementation-dependent
   - May rely on cloud provider features (S3 versioning)
   - Recovery scenarios if metadata corruption occurs need consideration

### Open Questions
1. **File Watcher Implementation**: How to efficiently detect and stage changed files using file watchers
   - Integration with daemon process
   - Handling of rapid file changes
   - Resource usage implications

2. **Database Growth**: Long-term strategy for metadata database size management
   - Retention policies for old snapshots
   - Archival strategies for ancient history
   - Acceptable for personal use but should monitor

3. **Concurrent Vault Writes**: If multiple hosts write same content simultaneously
   - Content is idempotent (safe)
   - Need protection mechanisms or just accept last-write-wins for rare edge cases?

## 4. Interface

The backup tool provides a command-line interface (CLI) called `bt`. The tool runs on a per-user basis on each host.

### Configuration Management

#### Initialize Configuration
```bash
bt config init
```
Creates configuration file in `~/.config/bt.toml` with vault settings (bucket, storage prefix, access credentials, etc.).

#### View Configuration
```bash
bt config list
```
Displays current configuration settings.

### Vault Management

#### Initialize Vault
```bash
bt vault init
```
Performs any necessary setup on the vault (e.g., creating bucket structure, verifying access).

### Directory Tracking

#### Track a Directory
```bash
bt init
```
Must be run within the directory to track. Marks this directory for backup without performing actual backup operations. Creates a Directory record in metadata.

### Backup Operations

#### Stage Files for Backup
```bash
bt add [FILENAME]
```
- Must be called within a tracked directory
- If FILENAME omitted, defaults to `.` (current directory)
- Scans and stages files for backup (synchronous operation)
- Calculates checksums and prepares operations in staging area (WAL)
- Does not upload to vault yet

#### Execute Backup
```bash
bt backup
```
- Host-level command (not scoped to current directory)
- Processes all staged operations from WAL
- Uploads content to vault
- Updates metadata database
- Uploads metadata database to vault
- Clears completed operations from WAL
- Can be run manually or by daemon process

### Status and Inspection

#### View Directory Status
```bash
bt status
```
Shows current state of files in the current tracked directory:
- Which files have been backed up
- Which files are staged but not yet backed up
- Which files are not tracked
- Can optionally show deleted files

#### View File History
```bash
bt log FILENAME
```
Shows version history for a specific file, including:
- Timestamps of each version
- Content checksums
- Metadata changes
- Which versions are available for restore

### Restore Operations

#### Restore a File
```bash
bt restore FILENAME [OPTIONS]
```
- Must be called within a tracked directory
- Restores file with different name (e.g., `filename.txt.<checksum>`)
- Options allow selecting specific version to restore
- By default restores full metadata (permissions, ownership, timestamps)
- Option to restore content only without metadata

### Background Operations (Future)

A daemon process will run in the background and:
- Periodically execute `bt backup` operations (e.g., every 15 minutes)
- Watch for file changes using file watchers (implementation TBD)
- Automatically stage changed files for backup

## 5. Design

### Data Model

The system uses the following core entities:

**Note on Identifiers:** All entity IDs are UUIDs except for Content.id, which is the checksum (SHA-256 or similar) of the content itself, enabling content-addressable storage.

#### Content
```
Content:
  - id: checksum (SHA-256 or similar) - not a UUID
  - created_at: timestamp (for bookkeeping)
  - (actual content stored in vault)
```
Represents immutable file content. The ID is the content's own checksum, enabling:
- Content-addressable storage
- Automatic deduplication across all hosts
- Integrity verification

#### Directory
```
Directory:
  - id: UUID
  - path: absolute path on host
  - created_at: timestamp
```
Represents a directory tracked for backup. Created when `bt init` is run. The directory exists within the context of the host, so no explicit host_id is needed.

#### File
```
File:
  - id: UUID
  - name: relative path within directory
  - directory_id: UUID (foreign key to Directory)
  - current_snapshot_id: UUID (foreign key to current FileSnapshot)
  - deleted: boolean flag
```
Represents a file within a tracked directory. The `name` is the relative path, so subdirectories are flattened (e.g., `subdir/file.txt`).

#### FileSnapshot
```
FileSnapshot:
  - id: UUID
  - file_id: UUID (foreign key to File)
  - content_id: checksum (foreign key to Content)
  - created_at: timestamp (when snapshot was created)
  - size: file size in bytes
  - permissions: file mode/permissions
  - uid: user ID
  - gid: group ID
  - accessed_at: file access time (atime from stat)
  - modified_at: file modification time (mtime from stat)
  - changed_at: metadata change time (ctime from stat)
  - born_at: file creation time (birthtime from stat, if available)
```
Represents the state of a file at a specific point in time. Each file has one current snapshot and a history of older snapshots.

### Storage Architecture

#### Vault Abstraction

**Configuration:**

```python
class VaultConfig:
    """
    Base configuration for vault implementations.
    Specific vault types should extend this with their own parameters.
    """
    def __init__(self, host_id: str):
        """
        Args:
            host_id: Unique identifier for this host (UUID)
        """
        self.host_id = host_id

class FileSystemVaultConfig(VaultConfig):
    """Configuration for filesystem-based vault."""
    def __init__(self, host_id: str, vault_root: Path):
        super().__init__(host_id)
        self.vault_root = vault_root

class S3VaultConfig(VaultConfig):
    """Configuration for S3-based vault."""
    def __init__(self, host_id: str, bucket: str, prefix: str, 
                 region: str, access_key: str, secret_key: str):
        super().__init__(host_id)
        self.bucket = bucket
        self.prefix = prefix
        self.region = region
        self.access_key = access_key
        self.secret_key = secret_key
```

**Vault Interface:**

The vault provides a storage abstraction with the following interface:

```python
class Vault:
    """
    Abstract interface for backup storage.
    All methods return Result indicating success/failure.
    File operations use paths to avoid loading large files into memory.
    """

    def __init__(self, config: VaultConfig):
        """
        Initialize vault with configuration.

        Args:
            config: VaultConfig or derived type with vault-specific settings
        """
        self.config = config

    def put_content(self, checksum: str, source_path: Path) -> Result:
        """
        Upload content-addressed object to vault.

        Args:
            checksum: Content identifier (SHA-256 or similar)
            source_path: Path to file containing content to upload

        Returns:
            Result indicating success/failure

        Behavior:
            - Idempotent: if content with this checksum already exists, 
              operation succeeds without uploading (no-op)
            - Checksum should be verified against actual content
        """
        pass

    def get_content(self, checksum: str, output_path: Path) -> Result:
        """
        Download content from vault by checksum.

        Args:
            checksum: Content identifier to retrieve
            output_path: Path where content should be written

        Returns:
            Result indicating success/failure

        Behavior:
            - Writes content to output_path
            - Should verify checksum after download
        """
        pass

    def put_metadata(self, host_id: str, source_path: Path) -> Result:
        """
        Upload host's metadata database to vault.

        Args:
            host_id: Unique identifier for this host
            source_path: Path to SQLite database file to upload

        Returns:
            Result indicating success/failure

        Behavior:
            - Overwrites existing metadata for this host_id
            - Implementation may use versioning to keep history
        """
        pass

    def get_metadata(self, host_id: str, output_path: Path) -> Result:
        """
        Download host's metadata database from vault.

        Args:
            host_id: Unique identifier for this host
            output_path: Path where database should be written

        Returns:
            Result indicating success/failure

        Behavior:
            - Writes most recent metadata to output_path
            - Returns error if no metadata exists for host_id
        """
        pass

    def init(self) -> Result:
        """
        Initialize vault structure (create buckets, verify access, etc.).

        Returns:
            Result indicating success/failure

        Behavior:
            - Idempotent: safe to call multiple times
            - Creates any necessary storage structures
            - Verifies credentials/permissions
        """
        pass
```

**Vault Implementations:**

```python
class FileSystemVault(Vault):
    """
    Local filesystem implementation for testing.

    Storage structure:
        <vault_root>/
            content/<checksum>           # Content objects
            metadata/<host_id>.db        # Metadata databases
    """
    def __init__(self, config: FileSystemVaultConfig):
        super().__init__(config)
        self.vault_root = config.vault_root
        self.content_dir = self.vault_root / "content"
        self.metadata_dir = self.vault_root / "metadata"

class S3Vault(Vault):
    """
    AWS S3 or S3-compatible storage implementation.

    Storage structure:
        s3://<bucket>/<prefix>/
            content/<checksum>           # Content objects
            metadata/<host_id>.db        # Metadata databases

    Features:
        - Can leverage S3 versioning for metadata history
        - Can use lifecycle policies for cold storage
        - Supports S3-compatible services (MinIO, Backblaze B2, etc.)
    """
    def __init__(self, config: S3VaultConfig):
        super().__init__(config)
        self.bucket = config.bucket
        self.prefix = config.prefix
        self.region = config.region
        # Initialize S3 client with credentials
```

#### Local State Management

**Configuration:**
- Location: `~/.config/bt.toml` (configurable)
- Contains: vault settings, credentials, local paths

**Metadata Database:**
- Location: `~/data/bt/db/metadata.db` (configurable)
- Format: SQLite database
- Contains: Directory, File, FileSnapshot, Content tables
- Updated locally during operations
- Uploaded to vault after successful backup

```python
class Database:
    """
    SQLite database interface for metadata management.
    Manages Directory, File, FileSnapshot, and Content records.
    """

    def __init__(self, db_path: Path):
        """
        Args:
            db_path: Path to SQLite database file
        """
        self.db_path = db_path

    # Core CRUD operations for entities
    # (Specific methods will be defined during implementation)
    # Examples:
    # - create_directory(path: Path) -> Directory
    # - get_file(directory_id: UUID, name: str) -> File
    # - create_file_snapshot(file_id: UUID, content_id: str, stats: FileStats) -> FileSnapshot
    # - update_file_current_snapshot(file_id: UUID, snapshot_id: UUID) -> Result
```

**Staging Area (WAL):**
- Location: `~/var/bt/wal/` (configurable)
- Format: Write-Ahead Log for pending operations
- Persists operations until completed
- Enables crash recovery

```python
class WALConfig:
    """Configuration for Write-Ahead Log."""
    def __init__(self, wal_path: Path):
        """
        Args:
            wal_path: Directory where WAL files are stored
        """
        self.wal_path = wal_path

class FileStats:
    """
    File metadata from stat system call.
    Corresponds to fields needed for FileSnapshot.
    """
    size: int                    # File size in bytes
    permissions: int             # File mode/permissions
    uid: int                     # User ID
    gid: int                     # Group ID
    accessed_at: timestamp       # Access time (atime)
    modified_at: timestamp       # Modification time (mtime)
    changed_at: timestamp        # Metadata change time (ctime)
    born_at: timestamp | None    # Creation time (birthtime, if available)

class WAL:
    """
    Write-Ahead Log for staging backup operations.
    Maintains a persistent queue of operations that must be completed.
    """

    def __init__(self, config: WALConfig, vault: Vault, database: Database):
        """
        Initialize WAL with configuration and dependencies.

        Args:
            config: WAL configuration
            vault: Vault instance for uploading content
            database: Database instance for metadata updates
        """
        self.config = config
        self.vault = vault
        self.database = database

    def enqueue_backup(self, file: File, staged_file_path: Path, 
                       checksum: str, file_stats: FileStats) -> Result:
        """
        Add a backup operation to the queue.

        Args:
            file: File object being backed up (contains id, directory_id, name, etc.)
            staged_file_path: Path to actual file content to upload
            checksum: Content checksum (will be used as content_id)
            file_stats: File metadata from stat

        Returns:
            Result indicating if operation was successfully enqueued

        Behavior:
            - Persists operation to disk immediately
            - Operation remains in queue until successfully processed
        """
        pass

    def stage_for_backup(self, directory: Directory, file: File) -> Result:
        """
        Stage a file for backup by copying to staging area and enqueuing operation.

        Args:
            directory: Directory object containing this file
            file: File object to stage for backup

        Returns:
            Result indicating success/failure

        Algorithm:
            1. Construct full file path from directory.path + file.name
            2. Get initial file stats (stat1)
            3. Copy file to staging area: <wal_path>/staging/<temp_uuid>
            4. Compute checksum of staged file
            5. Get final file stats (stat2) from original file path
            6. Verify file was not modified during copy:
               - Only atime should differ between stat1 and stat2
               - If other attributes changed, return error (file modified during backup)
            7. Call enqueue_backup(file, staged_path, checksum, stat2)

        Note:
            - Staged file is cleaned up after process_next_backup() completes
            - Uses temporary UUID filename until checksum is computed
        """
        pass

    def process_next_backup(self) -> Result:
        """
        Process the next backup operation in the queue.

        Returns:
            Result with one of:
                - Success: operation completed successfully
                - Failure: operation failed (remains in queue for retry)
                - NoWork: queue is empty

        Behavior:
            For the next operation in queue:
            1. Upload content to vault: vault.put_content(checksum, staged_file_path)
            2. Create FileSnapshot in database with file_stats
            3. Update File.current_snapshot_id to new snapshot
            4. Remove staged file from <wal_path>/staging/
            5. Dequeue operation from WAL

            Steps 4-5 only happen if steps 1-3 all succeed.
            If any step fails, operation remains in queue for retry.
        """
        pass

    def is_staged(self, file: File) -> bool:
        """
        Check if a file has a pending operation in the WAL queue.

        Args:
            file: File object to check

        Returns:
            True if file has pending backup operation, False otherwise
        """
        pass
```

**Service Layer:**

```python
class FileStatus(Enum):
    """Status of a file in the backup system."""
    IGNORED = "ignored"          # File matches ignore patterns
    UNTRACKED = "untracked"      # File not in database or never backed up
    STAGED = "staged"            # File staged in WAL, pending backup
    MODIFIED = "modified"        # File backed up but modified since last backup
    BACKED_UP = "backed_up"      # File backed up and unchanged
    DELETED = "deleted"          # File was backed up but no longer exists in filesystem

class BtService:
    """
    High-level service orchestrating backup operations.
    Implements business logic for CLI commands.
    """

    def __init__(self, database: Database, wal: WAL, vault: Vault):
        """
        Initialize service with database, WAL, and vault.

        Args:
            database: Database instance for metadata operations
            wal: WAL instance for staging and processing backups
            vault: Vault instance for content storage/retrieval
        """
        self.database = database
        self.wal = wal
        self.vault = vault

    def find_directory_for_path(self, path: Path) -> Directory | None:
        """
        Find the Directory that should contain this path.
        Checks the path itself and all parent directories.

        Args:
            path: Absolute path to search for

        Returns:
            Directory object if found, None otherwise

        Algorithm:
            1. Check if path itself is a tracked directory
            2. Check each parent directory in order
            3. Return first match, or None if no match found
        """
        # Check the path itself
        directory = self.database.find_directory_by_path(path)
        if directory:
            return directory

        # Check parent directories
        for dirname in path.parents:
            directory = self.database.find_directory_by_path(dirname)
            if directory:
                return directory

        return None

    def resolve_path(self, path: Path) -> tuple[Directory | None, File | None]:
        """
        Resolve an absolute path to its Directory and File records.

        Args:
            path: Absolute path to file

        Returns:
            Tuple of (directory, file) where either may be None
            - (None, None): No tracked directory contains this path
            - (directory, None): Path is in tracked directory but file not in database
            - (directory, file): Both directory and file found

        Algorithm:
            1. Find containing directory: directory = find_directory_for_path(path.parent)
            2. If no directory found, return (None, None)
            3. Calculate relative path: relative_path = path.relative_to(directory.path)
            4. Look up file: file = database.get_file(directory, relative_path)
            5. Return (directory, file)

        Note:
            - Assumes path is absolute
            - File may be None if not yet tracked in database
        """
        pass

    def should_ignore(self, directory: Directory, path: Path) -> bool:
        """
        Determine if a file should be ignored based on configuration.

        Args:
            directory: Directory containing the file
            path: Absolute path to file

        Returns:
            True if file should be ignored, False otherwise

        Note:
            Implementation will check:
            - Global ignore patterns from config
            - Local .btignore files (similar to .gitignore)
            - Details to be determined during implementation
        """
        pass

    def add_directory(self, path: Path) -> Result:
        """
        Track a directory for backup (implements `bt init`).

        Args:
            path: Absolute path to directory to track

        Returns:
            Result indicating success/failure

        Algorithm:
            1. Check permissions: user must have at least r+x on directory
               - If insufficient permissions, return failure

            2. Check if directory or any parent is already tracked:
               - Use find_directory_for_path(path)
               - If found, return success (no-op)

            3. Begin database transaction:
               a. Create directory: directory = database.create_directory(path)

               b. Consolidate any previously-tracked subdirectories:
                  - Find all subdirs: database.find_directories_by_path_prefix(path)
                  - For each subdir:
                      - Move files: database.move_files(source_dir=subdir, dest_dir=directory)
                        (Updates File.name to include relative path and File.directory_id to new directory)
                      - Delete old directory: database.delete_directory(subdir)

               c. Commit transaction

            4. Return success

        Note:
            - Operation is idempotent
            - All database operations are within a transaction
            - File snapshots and content remain unchanged during consolidation
        """
        pass

    def stage_file(self, path: Path) -> Result:
        """
        Stage a file for backup (implements `bt add filename`).

        Args:
            path: Absolute path to file (fully resolved by CLI)

        Returns:
            Result indicating success/failure

        Algorithm:
            1. Verify path is a regular file (not directory, symlink, etc.):
               - If not a regular file, return failure

            2. Find containing directory:
               - directory = find_directory_for_path(path.parent)
               - If not found, return failure (directory not tracked)

            3. Check if file should be ignored:
               - If should_ignore(directory, path), return success (no-op)
               - Respects global config and .btignore files (implementation detail)

            4. Get or create File record:
               - file = database.find_or_create_file(directory, path)
               - File.name is relative path from directory

            5. Stage for backup:
               - return wal.stage_for_backup(directory, file)

        Note:
            - Assumes path is fully resolved (no relative paths like '.')
            - Only handles regular files (not directories or symlinks)
            - Idempotent: safe to call multiple times for same file
        """
        pass

    def file_modified_since_backup(self, directory: Directory, file: File) -> bool:
        """
        Check if file has been modified since last backup.

        Args:
            directory: Directory containing the file
            file: File object to check

        Returns:
            True if file has been modified since last backup, False otherwise

        Note:
            Compares current filesystem metadata with file.current_snapshot
            Implementation details to be determined
        """
        pass

    def get_file_status(self, directory: Directory, relative_path: str) -> FileStatus:
        """
        Get the backup status of a specific file.

        Args:
            directory: Directory containing the file
            relative_path: Path relative to directory

        Returns:
            FileStatus enum value

        Algorithm:
            1. Check if file should be ignored:
               - full_path = directory.path / relative_path
               - If should_ignore(directory, full_path), return FileStatus.IGNORED

            2. Get file record:
               - file = database.get_file(directory, relative_path)
               - If not found, return FileStatus.UNTRACKED

            3. Check if file has been modified since backup:
               - If file_modified_since_backup(directory, file), return FileStatus.MODIFIED
               - (MODIFIED takes precedence over STAGED)

            4. Check if file is staged for backup:
               - If wal.is_staged(file), return FileStatus.STAGED

            5. Check if file has been backed up:
               - If not file.current_snapshot_id, return FileStatus.UNTRACKED
               - (File record exists but first backup never completed)

            6. Return FileStatus.BACKED_UP

        Note:
            - FileStatus.DELETED not applicable here (for files that exist in filesystem)
            - Precedence: IGNORED > MODIFIED > STAGED > BACKED_UP/UNTRACKED
        """
        pass

    def get_directory_status(self, path: Path) -> DirectoryStatus:
        """
        Get status of tracked directory (implements `bt status`).

        Args:
            path: Path to tracked directory

        Returns:
            DirectoryStatus with information about backed up, staged, and untracked files

        Note:
            Implementation will traverse filesystem and call get_file_status for each file.
            DirectoryStatus structure to be defined during implementation.
        """
        pass

    def get_file_history(self, path: Path) -> FileHistory | None:
        """
        Get version history for a file (implements `bt log filename`).

        Args:
            path: Absolute path to file

        Returns:
            FileHistory with list of snapshots and their metadata, or None if not found

        Algorithm:
            1. Resolve path to directory and file:
               - directory, file = resolve_path(path)
            2. If directory or file not found, return None
            3. Get file snapshots:
               - return database.get_file_snapshots_for_file(file, order_by="created_at")

        Note:
            - Returns None if file is not in a tracked directory or not in database
            - FileHistory structure to be defined during implementation
        """
        pass

    def restore_file(self, path: Path, checksum: str) -> Result:
        """
        Restore a file from backup (implements `bt restore filename --checksum=<checksum>`).

        Args:
            path: Absolute path to file
            checksum: Content checksum identifying which version to restore

        Returns:
            Result indicating success/failure

        Algorithm:
            1. Resolve path to directory and file:
               - directory, file = resolve_path(path)
            2. If directory or file not found, return failure
            3. Find file snapshot by checksum:
               - file_snapshot = database.find_file_snapshot_by_checksum(file, checksum)
            4. If file_snapshot not found, return failure
            5. Restore the file:
               - return restore_file_at(file_snapshot, path)

        Note:
            - Restored file will have a name derived from checksum (e.g., filename.txt.<checksum>)
            - Additional capabilities (restore by timestamp, metadata control) can be added later
        """
        pass

    def restore_file_at(self, file_snapshot: FileSnapshot, original_path: Path) -> Result:
        """
        Restore a specific file snapshot to disk.

        Args:
            file_snapshot: FileSnapshot to restore
            original_path: Original path of the file (used to derive restore path)

        Returns:
            Result indicating success/failure

        Algorithm:
            1. Construct restore path:
               - restore_path = original_path.parent / f"{original_path.name}.{file_snapshot.content_id}"
            2. Download content from vault:
               - result = vault.get_content(file_snapshot.content_id, restore_path)
            3. If download fails, return failure
            4. Restore file metadata:
               - Set permissions to file_snapshot.permissions
               - Set uid/gid to file_snapshot.uid/gid
               - Set timestamps (mtime to file_snapshot.modified_at, etc.)
            5. Return success

        Note:
            - Creates file with checksum suffix to avoid overwriting original
            - Metadata restoration may fail if user lacks permissions (acceptable)
        """
        pass
```

### Operation Flow

#### Adding Files (Staging)
1. User runs `bt add` in a tracked directory
2. System scans files/directory:
   - Calculate checksums for file contents
   - Gather filesystem metadata
   - Compare with current known state in local DB
3. For each changed/new file, create operation in WAL:
   - Operation includes: upload content, create/update FileSnapshot, update File
4. Operation is atomic unit that must fully succeed

#### Executing Backup
1. User runs `bt backup` (or daemon triggers it)
2. System processes WAL entries in order:
   - For each operation:
     a. Upload content to vault using `vault.put_content(checksum, file_path)` (idempotent if already present)
     b. Update local SQLite database (create FileSnapshot, update File.current_snapshot_id)
     c. Mark operation complete in WAL
3. After all operations complete:
   - Upload local SQLite database to vault using `vault.put_metadata(host_id, db_path)`
   - Clear completed WAL entries

#### Handling Failures
- If crash occurs during operation: resume from next incomplete WAL entry
- Content uploads are idempotent (duplicate checksums safely ignored)
- Each operation is atomic: either fully completes or remains in WAL
- Lost metadata updates are acceptable (download latest DB from vault and continue)

#### Querying History
To show file history:
1. Query FileSnapshots for given file_id, ordered by created_at
2. Display snapshot metadata and content checksums

To show directory state at point in time:
1. Query all Files in directory
2. For each File, get latest FileSnapshot where created_at <= target_time
3. Files may have slightly different backup times (acceptable)

#### Restoring Files
1. User runs `bt restore filename [--version=<timestamp>]`
2. System queries FileSnapshot:
   - If version specified: get snapshot at that created_at timestamp
   - Otherwise: get current snapshot
3. Download content from vault using `vault.get_content(content_id, output_path)`
4. Write to filesystem with modified name (e.g., `filename.<checksum>`)
5. Optionally restore metadata (permissions, timestamps, ownership)

### Multi-Host Coordination

**Content Sharing:**
- Multiple hosts can back up to same vault
- Content deduplication happens automatically (shared checksums)
- Each host has independent metadata store

**Metadata Isolation:**
- Each host maintains separate SQLite database
- Database identified by unique host_id
- No cross-host metadata visibility through this tool
- User can manually examine vault to see other hosts' data

**Host Identification:**
- host_id auto-generated on first run (UUID)
- Stored in local configuration
- Used to namespace metadata in vault

### Concurrent Operations

**Within Host:**
- Only daemon should write to metadata database
- CLI commands may read potentially stale data (acceptable)
- WAL provides operation serialization

**Across Hosts:**
- Content writes are idempotent (safe)
- Metadata is isolated per host (no conflicts)
- No explicit coordination needed

### Future Considerations

**Encryption:**
- Design accommodates future end-to-end encryption
- Could encrypt content before upload
- Metadata could be encrypted in vault
- Keys managed locally per host

**Incremental Backups:**
- Current model tracks all file versions
- Could be extended to delta-based storage
- Content model already supports this (just store deltas as Content)

**File Watching:**
- Daemon could use inotify/FSEvents to detect changes
- Automatically run staging for changed files
- Requires careful handling of rapid changes

## 6. Project Management

The project will be developed in phases, each delivering a testable, functional subset of features.

### Phase 1: Core Data Model and Local Operations
**Goal:** Establish data model and local metadata management without vault integration.

**Deliverables:**
- Define SQLite schema (Directory, File, FileSnapshot, Content reference)
- Implement local database operations (CRUD)
- Create configuration management (`bt config init`, `bt config list`)
- Implement directory tracking (`bt init`)
- Implement file scanning with checksum calculation
- Create WAL staging mechanism

**Testing:**
- Unit tests for database operations
- Test configuration management
- Test file scanning and checksum calculation
- Test WAL write and read operations

**Success Criteria:**
- Can initialize config and track directories
- Can scan files and calculate checksums
- Can stage operations in WAL
- All operations work with local SQLite database

### Phase 2: FileSystem Vault Implementation
**Goal:** Implement basic vault abstraction using local filesystem for testing.

**Deliverables:**
- Define Vault trait/interface
- Implement FileSystemVault (local directory as storage)
- Implement content upload/download (put_content, get_content)
- Implement metadata upload/download (put_metadata, get_metadata)
- Create `bt vault init` command for vault setup

**Testing:**
- Test content storage and retrieval
- Test metadata storage and retrieval
- Test idempotent content writes
- Integration test with local filesystem vault

**Success Criteria:**
- Can store and retrieve content by checksum
- Can store and retrieve metadata database
- Vault operations are idempotent
- Can initialize vault structure

### Phase 3: Backup Operations
**Goal:** Implement end-to-end backup flow from staging to vault.

**Deliverables:**
- Implement `bt add` command (file scanning and staging)
- Implement `bt backup` command (process WAL, upload to vault)
- Implement operation execution loop
- Implement crash recovery (resume from WAL)
- Handle file changes, new files, and metadata updates

**Testing:**
- Test add operation for single file
- Test add operation for directory
- Test backup execution
- Test crash recovery (interrupt and resume)
- Test duplicate content handling

**Success Criteria:**
- Can stage and execute backup of single file
- Can stage and execute backup of entire directory
- System recovers gracefully from crashes
- Content deduplication works correctly

### Phase 4: Status and Query Operations
**Goal:** Enable users to inspect backup state and history.

**Deliverables:**
- Implement `bt status` command (show directory state)
- Implement `bt log` command (show file history)
- Format and display file metadata
- Show staged vs backed up vs untracked files
- Display file version history with timestamps

**Testing:**
- Test status display for various directory states
- Test log display for files with multiple versions
- Test handling of deleted files
- Test display of staged operations

**Success Criteria:**
- Users can see current state of tracked directories
- Users can see version history for files
- Clear indication of what's backed up vs staged

### Phase 5: Restore Operations
**Goal:** Enable users to restore files from backup.

**Deliverables:**
- Implement `bt restore` command
- Support version selection (specific timestamp)
- Restore content with alternate filename
- Optionally restore metadata (permissions, ownership, timestamps)
- Handle missing content errors gracefully

**Testing:**
- Test restore of current version
- Test restore of specific historical version
- Test restore with and without metadata
- Test error handling for missing content

**Success Criteria:**
- Can restore any version of a backed-up file
- Restored files have correct content and metadata
- Original files are not overwritten
- Clear error messages for failures

### Phase 6: S3 Vault Implementation
**Goal:** Implement production-ready vault using AWS S3.

**Deliverables:**
- Implement S3Vault using vault interface
- Handle S3 authentication and configuration
- Implement retry logic for network failures
- Handle S3-specific errors gracefully
- Support S3-compatible services (optional)

**Testing:**
- Test against real S3 bucket
- Test with S3-compatible services (MinIO, etc.)
- Test error handling and retries
- Test with large files
- Performance testing

**Success Criteria:**
- Can back up to and restore from S3
- Handles network failures gracefully
- Works with S3-compatible services
- Performance is acceptable for personal use

### Phase 7: Daemon and Automation
**Goal:** Enable automated background backups.

**Deliverables:**
- Implement daemon process
- Periodic backup execution (configurable interval)
- Daemon lifecycle management (start, stop, status)
- Logging and error reporting
- (Stretch) File watcher integration for automatic staging

**Testing:**
- Test daemon startup and shutdown
- Test periodic backup execution
- Test daemon behavior during errors
- Test resource usage over time

**Success Criteria:**
- Daemon runs reliably in background
- Backups execute on schedule
- Errors are logged and reported
- System resource usage is reasonable

### Phase 8: Polish and Robustness
**Goal:** Improve user experience and system reliability.

**Deliverables:**
- Comprehensive error messages
- Progress indicators for long operations
- Better handling of large directories
- Configuration validation
- Documentation (README, usage guide)
- Handle edge cases (very long filenames, special characters, etc.)

**Testing:**
- User acceptance testing
- Edge case testing
- Documentation review
- Performance testing with large datasets

**Success Criteria:**
- Tool is ready for real-world personal use
- Clear documentation for all features
- Handles edge cases gracefully
- Performance is acceptable for typical use

### Future Phases (Post-MVP)
- End-to-end encryption
- Compression
- Incremental database syncing to vault
- File watcher-based automatic staging
- Cross-platform support (if not already)
- Additional vault implementations
- Retention policies and history management
- Large file chunking strategy

## Notes

This design document represents the initial architecture and is expected to evolve as implementation reveals new requirements or challenges. The phased approach allows for iterative development with frequent testing and validation.

Key principles maintained throughout:
- Simplicity over complexity
- Robustness for real-world use
- Language-agnostic design
- Testability at each phase
- Clear separation of concerns (vault abstraction, data model,
  operations)
