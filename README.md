
# **ASM (Automatic Storage Management) 

---

## **1. Introduction to ASM**

**Definition:**
Automatic Storage Management (ASM) is Oracle’s **integrated storage management system** that automates database file placement, striping, and mirroring.

**Notes:**

1. Acts as a **file system + volume manager** for Oracle databases.
2. Eliminates the need to manage thousands of individual database files.
3. DBA only manages **disk groups**, ASM manages file placement automatically.
4. ASM **replaces the need for a separate volume manager**, as it can handle striping, mirroring, and space allocation natively.

**Example:** Instead of managing 5,000+ files and using LVM or RAID separately, you just create 2–3 disk groups (`+DATA`, `+FRA`) and ASM handles everything.

---

## **2. ASM as a Volume Manager**

**Definition:**
ASM functions as a **volume manager**, meaning it manages disks and storage logically, abstracts them from the OS, and controls allocation, striping, and mirroring of files.

**Notes:**

1. Disks are grouped into **disk groups** which ASM treats as a single logical storage unit.
2. ASM handles **allocation, placement, striping, mirroring**, and **rebalancing** automatically.
3. Eliminates need for OS-level LVM or RAID for Oracle datafiles, though ASM can use RAID provided by the storage system.
4. ASM manages **failure groups**, ensuring redundancy and high availability.

**Example:**

* Physical disks `/dev/sd1, /dev/sd2, /dev/sd3` → ASM disk group `+DATA`
* ASM automatically stripes and mirrors files across disks → No manual volume management needed.

---

## **3. ASM Architecture**

**Definition:**
ASM architecture consists of a **database instance** and a **special ASM instance** that manages storage for Oracle databases.

**Notes:**

1. **Database Instance** → Runs SQL and stores data in files.
2. **ASM Instance** → Manages disk groups, file placement, striping, and mirroring.
3. ASM instance is lightweight with its own **SGA & background processes**.
4. Does **not store user data**, only metadata.
5. In RAC, each node has one ASM instance; they communicate peer-to-peer.

---

## **4. Disk Groups**

**Definition:**
A disk group is a **collection of disks** managed as a single logical unit in ASM.

**Notes:**

1. All database files are stored in disk groups.
2. ASM spreads files across disks using **striping** and **mirroring**.
3. Multiple disk groups can exist: e.g., `+DATA` (datafiles), `+FRA` (backup/archivelogs).

**Example:**

```sql
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY
DISK '/dev/sd1', '/dev/sd2';
```

---

## **5. ASM Benefits**

### **5.1 Striping**

**Definition:** Splitting data into equal-sized chunks and distributing them across all disks in a disk group.

**Purpose:** Improves performance and balances I/O.

**Example:** 100 MB file → 25 MB stored on each of 4 disks.

---

### **5.2 Mirroring**

**Definition:** Creating redundant copies of data at the **file extent level** for fault tolerance.

**Purpose:** Protects against disk failures.

**Options:**

* **2-way mirroring** → one original + one copy
* **3-way mirroring** → one original + two copies

**Example:**

```sql
CREATE DISKGROUP FRA NORMAL REDUNDANCY
DISK '/dev/sd3', '/dev/sd4', '/dev/sd5';
```

---

### **5.3 Online Reconfiguration & Rebalancing**

**Definition:** ASM can add or remove disks **while the database is running**, and automatically redistributes data evenly.

**Purpose:** Maintains performance and space balance without downtime.

**Examples:**

```sql
ALTER DISKGROUP DATA ADD DISK '/dev/sd6';
ALTER DISKGROUP DATA DROP DISK '/dev/sd2';
```

**Monitor Rebalancing:**

```sql
SELECT GROUP_NUMBER, FILE_NUMBER, STATE, PHASE
FROM V$ASM_OPERATION;
```

---

## **6. ASM Disk Types (Header Status)**

**Definition:** Disk type shows the **status of a disk** in ASM.

| Disk Type       | Meaning                                       | Notes                                          |
| --------------- | --------------------------------------------- | ---------------------------------------------- |
| **CANDIDATE**   | Disk not part of any disk group, can be added | Fresh disk ready for ASM                       |
| **PROVISIONED** | Disk prepared at OS level, not added yet      | Extra setup done (zoning, multipathing)        |
| **MEMBER**      | Already part of a disk group                  | Cannot be added to another group unless forced |
| **FORMER**      | Previously part of a disk group, now removed  | Can be reused in a new disk group              |

**Check Disk Status:**

```sql
SELECT NAME, PATH, HEADER_STATUS
FROM V$ASM_DISK;
```

---

## **7. ASM Redundancy Levels**

**Definition:** Redundancy defines how ASM protects database files using mirroring.

| Redundancy   | Description                                         | Failure Tolerance                                    |
| ------------ | --------------------------------------------------- | ---------------------------------------------------- |
| **EXTERNAL** | No ASM mirroring; storage system handles protection | Depends on hardware RAID                             |
| **NORMAL**   | 2-way mirroring                                     | Survives 1 disk failure                              |
| **HIGH**     | 3-way mirroring                                     | Survives 2 disk failures in different failure groups |

**Example:**

```sql
CREATE DISKGROUP CRIT HIGH REDUNDANCY
DISK '/dev/sd6', '/dev/sd7', '/dev/sd8', '/dev/sd9';
```

---

## **8. ASM Instance Processes**

**Definition:** Background processes in ASM that manage storage, rebalancing, and disk health.

| Process   | Function                                             |
| --------- | ---------------------------------------------------- |
| **RBAL**  | Coordinates rebalancing when disks are added/removed |
| **ARBn**  | Moves data during rebalancing                        |
| **GMON**  | Monitors disk group membership and failure groups    |
| **MARK**  | Marks bad or stale allocation units                  |
| **RBALn** | Worker processes performing data movement            |

**Notes:**

* RBAL + ARBn handle **all rebalancing automatically**.
* GMON ensures **disk group integrity**.
* MARK identifies **bad extents**.
* Run **only in ASM instance**, not database instance.

---

## **9. ASM Administration**

**Definition:** Methods and tools used to manage ASM.

**Tools:**

1. **Oracle Enterprise Manager (OEM)**
2. **SQL\*Plus**
3. **ASMCMD** – command-line tool for ASM

**ASMCMD Examples:**

```bash
ASMCMD> lsdg      -- List disk groups
ASMCMD> ls        -- List files
ASMCMD> du        -- Show space usage
ASMCMD> rm        -- Remove file
```

---

## **10. ASM File Management**

**Definition:** ASM stores almost all Oracle database files and manages their placement.

**Files Managed by ASM:**

1. Datafiles
2. Redo logs
3. Control files
4. SPFILE
5. Tempfiles

**Examples:**

**Create tablespace in ASM:**

```sql
CREATE TABLESPACE users
DATAFILE '+DATA'
SIZE 100M AUTOEXTEND ON;
```

**Add redo logs to ASM:**

```sql
ALTER DATABASE ADD LOGFILE '+FRA' SIZE 200M;
```

---

## **11. Quick Recap**

1. ASM = Oracle automatic storage manager.
2. Functions as **file system + volume manager**.
3. Disk Groups = logical storage units.
4. Benefits = **Striping, Mirroring, Rebalancing**.
5. Disk Types = Candidate, Provisioned, Member, Former.
6. Redundancy = External, Normal, High.
7. ASM Instance = lightweight, manages metadata.
8. ASM Processes = **RBAL, ARBn, GMON, MARK, RBALn**.
9. Admin Tools = OEM, SQL\*Plus, ASMCMD.
10. Stores = Datafiles, Control files, Redo logs, Tempfiles, SPFILE.

---

