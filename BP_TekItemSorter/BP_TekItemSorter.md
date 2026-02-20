> ## Note
> #### This documentation was created with the assistance of AI, based on my descriptions of the systems during development, it cleaned, spell corrected and organized my chaotic thought process. While care has been taken to ensure accuracy, AI is known to hallucinate and it did, several times... actually a bunch of times... more times than is necessary.
>
>#### Its primary use is intended for future reference and to aid understanding of the system architecture and keep me on track with its development. This project was supposed to be simple, but the universe decided otherwise.
>
>#### This documentation is meant to capture the intent and overall design of the system, making it easier to revisit, maintain, or extend in the future, as well as helping others understand how the pieces fit together. There will be many function, variable and event signatures mentioned while having no concrete definition shown in this document, this is because the size and scope of the project makes this time prohibitive. 
>
> I've used VB-like syntax as a structured 'pseudocode' because it is easy for laypeople to understand.. mostly. 
> I usually prototype in C++ but that is not so obvious to people who have never used code in their lives.
>
> ### Because I suck at documentation .. Enjoy.

---
## Table of Contents

- [01. Global Architecture](#i-global-architecture)
- [02. The Thermal Regulation Subsystem (`AC_HeatSystem`)](#ii-the-thermal-regulation-subsystem-ac_heatsystem)
  - [Biome Awareness](#biome-awareness)
  - [Internal Procedural Logic](#1-internal-procedural-logic)
    - [Passive Dissipation (convective cooling)](#passive-dissipation-convective-cooling)
    - [Liquid Cooling (active convective cooling)](#liquid-cooling-active-convective-cooling)
    - [Boost Cooling (particle electron plasma cooling)](#boost-cooling-particle-electron-plasma-cooling)
    - [Why this feels “real”](#why-this-feels-real)
  - [Purpose and Benefits](#3-purpose-and-benefits)
- [03. Task MetaData: The Architectural Anchor (`F_ItemSorterTask_MetaData`)](#iii-task-metadata-the-architectural-anchor-f_itemsortertask_metadata)
  - [Internal Procedural Logic](#internal-procedural-logic)
  - [Purpose and Benefits](#purpose-and-benefits)
  - [Integration Examples](#integration-examples)
- [04. The Task Framework (`AC_ItemSorter_Task_Base`)](#iv-the-task-framework-ac_itemsorter_task_base)
  - [Non-Ticking, Event-Driven Architecture](#non-ticking-event-driven-architecture)
  - [The Three-Gate Check](#the-three-gate-check)
  - [Purpose and Benefits](#purpose-and-benefits-1)
  - [Logistics Optimisation: Inventory-Aware Routing](#logistics-optimisation-inventory-aware-routing)
- [05. The Butler (Maintenance) Subsystem (`AC_ItemSorter_Task_Butler`)](#v-the-butler-maintenance-subsystem-ac_itemsorter_task_butler)
  - [Reasoning: Soft Object References](#reasoning-soft-object-references)
- [06. The Custom Task Subsystem (`AC_ItemSorter_Task_Custom`)](#vi-the-custom-task-subsystem-ac_itemsorter_task_custom)
- [07. The Area Scan Subsystem (`AC_AreaScanManager`) Component](#vii-the-area-scan-subsystem-ac_areascanmanager-component)
  - [Design Intent & Purpose](#1-design-intent--purpose)
  - [Internal Procedural Logic (Lazy Scan Resolution)](#2-internal-procedural-logic-lazy-scan-resolution)
  - [Interface Definition (`BI_AreaScanManager`)](#3-interface-definition-bi_areascanmanager)
  - [Benefits and Performance Impact](#5-benefits-and-performance-impact)
- [08. UI & List Management](#viii-ui--list-management)
- [09. Communication & Safety (Interfaces)](#ix-communication--safety-interfaces)
- [10. Status: Subsystems in Progress](#x-status-subsystems-in-progress)
- [11. Passing Ideas that may get implemented](#xi-passing-ideas-that-may-get-implemented)

---
## BP_TekItemSorter: Technical Architecture & Subsystem Specification

This document provides a technical breakdown of the **BP_TekItemSorter** architecture and its integrated subsystems. This system is designed as a high-performance, server-authoritative logistics solution for ARK: Survival Ascended (ASA), prioritising modularity and server stability. There is much more to this than what is documented here as mentioned in the opening note, as even with AI helping with the documentation, it would take as much time to document everything in fine detail as it would to actually develop the thing. maybe i'll flesh out the missing bits at some point if i feel the context is warranted, but I'm notoriously bad at documentation. The main thing is, the core architecture is documented.

Here we go...

---

### 01. Global Architecture
The system is built on a **Base-Child Actor Component** framework. The primary actor, `BP_TekItemSorter`, acts as a host for specialised components that handle specific logistics logic.

* **Constraint:** By default tribe is strictly limited to **two** `BP_TekItemSorter` structures.
* **Purpose:** This limit prevents "automation bloat" where a single tribe could potentially stall a server thread by running thousands of simultaneous inventory scans.
* **Benefit:** It forces players to be strategic with their logistics and ensures that the Heat System remains a meaningful gameplay mechanic rather than something easily bypassed by building more structures.

---

## 02. The Thermal Regulation Subsystem (`AC_HeatSystem`)

The `AC_HeatSystem` is the "Governor" of the structure. It regulates automated task throughput to ensure the system feels physical, predictable, and self-balancing. Instead of arbitrary hard limits, the system generates **Heat** as it performs work, mirroring real-world automation where throughput is limited by thermal constraints.

#### **Biome Awareness:** 
The `BaseDissipationRate` isn't static. It is modified by the `AmbientTemperature` of the Biome the Sorter is placed in.

**Snow Biome:** Higher passive dissipation, but Water-based Liquid Cooling has a higher risk of "Freezing".

**Volcano/Desert** Biome: Lower passive dissipation, making Liquid/Boost cooling mandatory for high-frequency tasks.
    
### Internal Procedural Logic

* **Heat Accumulation:** Whenever a task component performs a meaningful step (e.g., World Scan, Resolve Source, Checks Inventory Capacity ...), it calls `AddHeat`. Heat is generated **incrementally** during execution, not once at the end, allowing for real-time heat gain and cooling interaction.
* **Scaling Philosophy:** To keep heat growth smooth, the "cost" of an action is scaled by its **Operation Category**, the **Task Frequency**, the **Quantity involved**, and a specific **Divisor** tuned to real-world usage patterns.
* **Throttling:** As heat exceeds the `HeatSoftCap`, the `PenaltyMultiplier` increases. This multiplier is consumed by Task Components to increase the delay between queue entries and increase the scheduling time.

The layered cooling design intentionally mirrors **physical principles** to make the system feel intuitive and “realistic” without exposing the full complexity of the underlying math to the player with overly complicated instructions in the user interface.

---

#### **Passive Dissipation (convective cooling)**
- **Mechanic:** Scales with current heat; high heat dissipates faster.
- **Real-world analogy:**  
  - Hot objects radiate energy faster than cooler objects.  
  - Convection is stronger when the temperature differential between object and environment is larger.  
- **Implementation:**  
  - `ProposedHeatReduction = ((CurrentHeat / HeatSoftCap) + 1) * BaseDissipationRate`  
  - This makes heat drop more rapidly when the structure is hot, then taper off naturally as it cools.

---

#### **Liquid Cooling (active convective cooling)**
- **Mechanic:** Consumes stored water or liquid resources to boost cooling, only above freezing (273.15K).  
- **Real-world analogy:**  
  - Water transfers heat efficiently via conduction and evaporation.  
  - Below freezing, water can’t flow, so it stops being effective.  
- **Implementation:**  
  - The system checks `CurrentHeat > 273.15` before applying liquid cooling.  
  - Liquid consumption scales proportionally to the required cooling (`ProposedHeatReduction * 1.1`) but is clamped to available reserves.

---

#### **Boost Cooling (particle electron plasma cooling)**
- **Mechanic:** Energy-based cooling works at any temperature, consuming Element Shards.  
- **Real-world analogy:**  
  - Theoretical electron plasma cooling, or exotic energy transfer, these systems work regardless of environmental temperature.  
  - They are very costly in energy, so overusing them is limited by resource availability.
- **Implementation:**  
  - Diminishing returns factor in, the closer the structure is to the ambient biome temperature.
  - The closer to absolute zero the structure is cooled, the more resources it will consume.
  - When at absolute zero, it will continue to consume available resources to maintain the low temperature.
  - If Energy-based Cooling is unavailable due to no resources, the temperature will return to the ambient biome temperature.

>Boost Cooling can theoretically drive the structure temperature all the way down to absolute zero (0 K / -273.15°C / -459.67°F), though in practice this is extremely resource-intensive. If boost resources are depleted, temperature will naturally drift back toward the current biome ambient temperature."

---

### **Why this feels “real”**
- High heat makes things “dangerous” and also naturally cools faster, mimicking real world physics.  
- Using water as a cooling medium is intuitive; freezing water doesn’t help in convection cooling.  
- Boost items act like a backup, high-cost, always-on cooling system, like thermoelectric systems or refrigerant.
- Layers combine to produce a smooth, predictable, and believable cooling curve that reacts to task load, environmental constraints, and player input.

---

<details>
<summary>>Code Structure (pseudo code)</summary>

```vbnet
' Categories representing why heat is generated
Public Enum E_TaskOperationType ' defaults
    Scan        ' 0.75
    Inspect     ' 0.75
    Validate    ' 0.25
    Evaluate    ' 0.10
    Resolve     ' 0.20
    Transfer    ' 1.75
    Executed    ' 1.50
    Time        ' 0.10
    Other       ' 1.00
End Enum

' Scaling factor for task friction
Public Enum E_TaskFrequency ' defaults
    VerySlow ' 0.25
    Slow     ' 0.50
    Normal   ' 1.00
    Fast     ' 2.00
    VeryFast ' 3.00
End Enum

Public Class AC_HeatSystem
    Private CurrentHeat As Double ' 0.0 K

    Private HeatSoftCap As Double = 273.15 ' Kelvin - not a stopping point, but a calculation trigger
    Private HeatHardCap As Double = 1811.15 ' Kelvin - not a stopping point, but a calculation trigger

    
    Private CoolingLiquid As Double ' g/m³ current storage
    Private CoolingBoost As Double ' g/m³ current storage
    Private GlobalHeatMultiplier As Double = 1.0

    Private CoolingLiquidMax As Double = 10000.0 ' g/m³
    Private CoolingLiquidCost As Double = 0.272 ' % base cost

    Private CoolingBoostMax As Double = 1000.0 ' g/m³ storage
    Private CoolingBoostCost As Double = 1.0 ' % base cost
    
    ' Operation-specific divisors (fixed magic numbers - tuned for balance)
    ' Higher divisor = less heat per unit quantity processed
    ' These are NOT currently server-configurable; only GlobalHeatMultiplier is exposed.
    ' Future configuration options may expose/adjust these.


    ' Incremental heat addition called during task execution steps.
    Public Function AddHeat(operationType As E_TaskOperationType, frequency As E_TaskFrequency, amount As Double)
        If Not HasAuthority() Then Exit Sub
        
        Dim operationCost As Double
        Dim divisor As Double

        ' Logic-based weighting of different work types
        Select Case operationType
            Case E_TaskOperationType.Scan : operationCost = 0.75 : divisor = 20.0
            Case E_TaskOperationType.Inspect : operationCost = 0.5 : divisor = 50.0
            Case E_TaskOperationType.Validate : operationCost = 1.2 : divisor = 15.0
            Case E_TaskOperationType.Evaluate : operationCost = 1.5 : divisor = 150.0
            Case E_TaskOperationType.Resolve : operationCost = 2.5 : divisor = 5.0
            Case E_TaskOperationType.Transfer : operationCost = 2.0 : divisor = 250.0
            Case E_TaskOperationType.Executed : operationCost = 0.1 : divisor = 1.0
            Case E_TaskOperationType.Time : operationCost = 3.0 : divisor = 5.0
            Case Else : operationCost = 3.5 : divisor = 50.0
        End Select

        ' Frequency scaling
        Dim frequencyFactor As Double = GetFrequencyFactor(frequency)

        Dim operationScale As Double = Math.Max(0.0, amount / divisor)
        Dim heatGain As Double = operationCost * operationScale * frequencyFactor * GlobalHeatMultiplier

        SetCurrentHeat(CurrentHeat + heatGain)
        Return heatGain
    End Sub


    ' Final calculation of cooling effects, consuming resources where applicable.
    Private Function GetCoolingFinal(proposedHeatReduction As Double) As Double
        If CurrentHeat <= 0 Then Return 0.0

        Dim coolingLiquidUsing As Double = 0.0
        Dim coolingBoostUsing As Double = 0.0
        
        ' Liquid cooling rewards infrastructure but requires > Freezing Temp
        If CoolingLiquid > 0 AndAlso CurrentHeat > 273.15 Then
            coolingLiquidUsing = proposedHeatReduction * 1.1
        End If

        ' Boost cooling works at any temperature for a resource cost
        coolingBoostUsing = Math.Min(CoolingBoost, (CoolingBoostMax * (CoolingBoostCost / 100.0)))

        '
        ' [Other very long math and `Double` curve calculations]
        '

        ' Apply resource usage and return total reduction
        UpdateResourceBuffers(coolingLiquidUsing, coolingBoostUsing)
        Return proposedHeatReduction + (coolingLiquidUsing * CoolingLiquidFactor) + (coolingBoostUsing * CoolingBoostFactor)
    End Function

    ' ... Other Getters & Setters ...

End Class
```

</details>

---

### Purpose and Benefits

* **Fairness:** Heat reflects real work done. Small, infrequent tasks feel light, while massive, high-speed logistics require infrastructure (Coolant/Boosters).
* **Predictability:** The Risk Level System (Low to Critical) provides visual foreshadowing of environmental danger, encouraging investment in cooling before the system throttles or causes damage.
---

### 03. Task MetaData: The Architectural Anchor (`F_ItemSorterTask_MetaData`)

The `F_ItemSorterTask_MetaData` is the most critical `Struct` in the project. It serves as the **"Standardised Header"** for every task, regardless of whether that task is a `Custom` (Many-to-Many) or `Butler` (Maintenance) type. 

By nesting this metadata at the top of every task-specific struct, we allow the **Base Component** to handle global logic (like scheduling, pausing, and identification) without needing to understand the unique "guts" of the specific task.

---

<details>
<summary>>Code Structure (pseudo code)</summary>

```vbnet

' The core header for all Item Sorter tasks. 
' Nested within F_ItemSorterTask_Butler and F_ItemSorterTask_Custom.
Public Structure F_ItemSorterTask_MetaData


    ' A unique string generated upon task creation (GUID).
    ' Used to bridge the UI, the Component, and the SaveGame.
    Public TaskUUID As String


    ' Player-defined label (e.g., "Main Base Metal Stocking").
    Public TaskName As String


    ' Defines if this is a Butler (Maintenance) or Custom (Logistics) task.
    Public TaskType As E_TekItemSorter_TaskType


    ' Linked to the Heat System Map to determine friction and timing.
    Public TaskFrequency As E_TekItemSorter_TaskFrequency


    ' Global toggle. If True, the task component is never added to the Item Sorter, if its active, its destroyed.
    Public IsTaskPaused As Boolean


    ' A flag to mark the status of the task. 
    Public TaskStatus As E_TaskStatusFlags


    ' The Unix Epoch (in milliseconds) for the next allowed run.
    Public ScheduledEpochMillis As Int64

End Structure
```

</details>

---

#### Internal Procedural Logic

The MetaData drives three core system behaviors:

1.  **Identification & Mapping:** The `TaskUUID` acts as the primary key. When the UI sends a command to Create, Read, Update, or Delete a task, the Sorter searches for the component or task entry that matches this UUID.
2.  **The Scheduling Gate:** The `ScheduledEpochMillis` is the server-authoritative timestamp. The Base Component's `Tick` or `Timer` logic performs a simple comparison: `If (CurrentServerTime >= ScheduledEpochMillis) Then Execute()`. 
3.  **UI Serialization:** When the Sorter replicates the list of active tasks to a player’s UI, it uses the MetaData to populate the "Common" fields (Name, Frequency, Status) before the task-specific widget (Butler vs. Custom) fills in the rest.
4.  **Event Capture:** Almost all task related events check against `TaskUUID`, on the UI, in the player attached `Buff_ItemSorter`, structure and components. The whole ecosystem depends on this identifier.
---

### Purpose and Benefits

* **Polymorphic Logic:** It allows the `AC_TekItemSorter_Task_Base` to treat all tasks as the same "Type" for scheduling purposes. This prevents code duplication.
* **Persistent Timing:** Because `ScheduledEpochMillis` is a `Long` (Int64), the "Next Run Time" survives server restarts. If a task was supposed to run at 12:00 PM and the server was down, it will trigger almost immediately upon the server coming back up or very soon after.
* **Performance:** The Sorter can quickly iterate through a list of MetaData to find a task name or UUID without having to load or "Unpack" the heavy data associated with task payloads.

---

### Integration Examples

#### In the Blueprint logic of `AC_TekItemSorter_Task_Base`, (The task parent):

1.  **Read MetaData:** The component pulls its internal `TaskMetaData` variable.
2.  **ConditionA:** `If (MetaData.TaskUUID == "")` -> **Abort.**
3.  **ConditionB:** `If (MetaData.IsTaskPaused == True)` -> **Abort.**
4.  **Execute:** If pass, call the child-specific `UpdateTransferQueue()` logic (overridden by Butler or Custom).
5.  **Reschedule:** Once work is done, the MetaData is updated: `MetaData.ScheduledEpochMillis = ServerTime + NewInterval`.

#### In the Blueprint logic of `UW_LI_ItemSorter_Entry_Custom`, (a ListView widget entry for a custom task):

1.  **Read MetaData:** The widget captures a dispatched event from the `Buff_ItemSorter` the widget reads the MetaData.
2.  **Condition A:** `If (MetaData.TaskUUID != LocalTaskMeta.LocalTaskUUID)` -> **Ignore**
4.  **Execute:** If pass, call the local function `UW_LI_ItemSorter_Entry_Custom.UpdateLocalMetaData(MetaData)`.
3.  **Action A:** `If (MetaData.IsTaskPaused != LocalTaskMeta.IsTaskPaused)` -> **Update LocalTaskMeta**
5.  **Action B:** `If (MetaData.TaskName != LocalTaskMeta.TaskName)` -> **Update LocalTaskMeta**
6.  **Done:** Once Event is handled, refresh rendered elements.

#### In the Blueprint logic of `Buff_ItemSorter` (player-attached buff used for Server ↔ Owning Client RPCs):

1. **Receive Request:** `Buff_ItemSorter` receives a Run on Server interface call from `U_ItemSorter_Main` indicating a task was deleted, passing `TaskMeta`.
2. **Relay to Sorter:** `Buff_ItemSorter` calls `BI_ItemSorter.IDoTask_Delete(TaskMeta)`.
3. **Execute:** `BP_ItemSorter` executes `DeleteTask(TaskMeta)`, then fires the dispatcher `TaskAction_Delete(Self, TaskUUID)`.
4. **Relay to Owning Client:** `Buff_ItemSorter`, subscribed to `TaskAction_Delete`, captures the event and calls the Run on Owning Client RPC `ROC_Action_Delete(TaskUUID)`.
5. **Client-Side Event:** Execution transitions to the owning client, where `TaskAction_Deleted(Self, TaskUUID)` is triggered.
6. **Update UI:** `U_ItemSorter_Main`, subscribed to `TaskAction_Deleted`, removes the UI task entry matching `TaskUUID`.



### 04.1. Replication Pipeline
This replication pipeline describes a Three-Tier State Management System designed to synchronize complex persistent data between a server-side world object and a client-side user interface. It avoids the performance pitfalls of constant "ticking" by using an event-driven "Bridge" architecture.

```ini
     [SERVER SIDE]                      [NETWORK]                    [CLIENT SIDE]
+-----------------------+       +-----------------------+      +-----------------------+
|  BP_TekItemSorter     |       |   Buff_ItemSorter     |      |   U_ItemSorter_Main   |
| (Structure Authority) | <===> |    (RPC Bridge)       | <===>|      (Client UI)      |
+-----------------------+       +-----------------------+      +-----------------------+
           ^
           |
           v
+-----------------------+
| Saved*TaskType*Tasks  | 
|  (SaveGame Array)     | 
+-----------------------+
```


---

### 04. The Task Framework (`AC_ItemSorter_Task_Base`)
All logistics tasks (Custom or Butler) inherit from this base component. It provides the standardised logic for scheduling and the execution of the `TransferQueue`.

#### Non-Ticking, Event-Driven Architecture

To maintain a negligible CPU footprint, `AC_ItemSorter_Task_Base` and its children are explicitly configured as Non-Ticking Components `(bCanEverTick = false)`.

The execution loop is entirely event-driven, relying on `TimerByEvent` handles managed by the Server’s task scheduler. This avoids the overhead of per-frame updates. All active timers are strictly cleared and invalidated during the component's `EndPlay` or through explicit interface calls (`IComponentTask_End`), ensuring no "zombie" logic remains in memory after a task is stopped or the actor is destroyed.

### The Three-Gate Check:
    Gate 1 (Identity): Does the TaskUUID exist in the Sorter’s ActiveTasks Map? (Prevents ghost components).
    Gate 2 (Energy): Is the Sorter powered and "Activated"? (Prevents free automation).
    Gate 3 (Range): Are the Source or Destination actors still within the ScanningRadius?

#### Purpose and Benefits
* **Decoupled Processing:** Each task component processes its own `TransferQueue` independently. This means a Butler task maintaining a stack of Element in a Tek generator does not have to wait for a massive Custom Task moving 250 Charge Batteries to finish its queue.
* **Heat-Driven Delay:** The Base component ensures that the time between moving `Index 0` and `Index 1` in the queue is scaled by the Sorter’s current heat from `AC_HeatSystem.GetEfficiencyMultiplier()`. This "Time-Slicing" ensures the server always has room to breathe between queue entries and item transfers.


---

<details>
<summary>>Code Structure (pseudo code)</summary>

```vbnet
' Standardised movement instruction generated by children for the base worker.
Public Structure F_TransferQueueEntry
    Public Property TargetPrimalItemClass As PrimalItem.Class
    Public Property SourcePrimalInventoryComponent As PrimalInventoryComponent
    Public Property DestinationPrimalInventoryComponent As PrimalInventoryComponent
    Public Property TargetDurabilityPercentage As Float
    Public Property DurabilityComparisonMethod As E_MathFunctions
End Structure

' Interface to manage the task on the result of UI actions
Public Interface BI_ItemSorter_Tasks
    Sub IComponentTask_End()
    Sub IComponentTask_Restart()
End Interface


' BASE CLASS: Manages Lifecycle, Validation, and Timers.
Public Class AC_ItemSorter_Task_Base
    Inherits UActorComponent
    Implements BI_ItemSorter_Tasks

    ' Variables
    Public Property ActiveTaskMetaData As F_ItemSorterTaskMetaData
    Public Property TransferQueue As Array(Of F_TransferQueueEntry)
    Public Property TaskWorkerTimerHandle As TimerHandle

    ' --- Lifecycle Control ---
    Protected Overrides Sub BeginPlay()
        If Not Me.GetOwner().HasAuthority() Then
            Me.DestroyComponent()
            Return
        End If

        MyBase.BeginPlay()

        ' 1. Centralised Validation Toll-Booth
        ' If UUID is invalid, destroy immediately before the loop starts
        If String.IsNullOrWhiteSpace(ActiveTaskMetaData.TaskUUID) Then
            Me.DestroyComponent()
            Return
        End If

        ' 2. Start the recursive loop
        Me.ScheduleTaskWorker()
    End Sub

    Private Sub ScheduleTaskWorker() ' Event

        ' Pause Check
        If ActiveTaskMetaData.IsTaskPaused Then
            Me.DestroyComponent()
            Return
        End If

        Dim ItemSorterInterface As BI_ItemSorter = GetOwner(self)

        If Not IsValidInterface(ItemSorterInterface) Then
            Me.DestroyComponent()
            Return
        End If

        If Not ItemSorterInterface.IsContainerActivated() Then
            Me.DestroyComponent()
            Return
        End If

        Dim RandomInFrequencyRange As Double = ItemSorterInterface.IGetIntervalFrequency(ActiveTaskMetaData.TaskFrequency)

        Dim HeatSystemInterface As BI_HeatSystem = ItemSorterInterface.IGetHeatSystem()

        Dim CurrentPenaltyMultiplier As Double = HeatSystemInterface.IGetPenaltyMultipliers().LogCurve

        Dim DelayTime As Double = RandomInFrequencyRange * CurrentPenaltyMultiplier

        ' Set the timer for the next worker execution
        TaskWorkerTimerHandle = KismetSystemLibrary.SetTimerByEvent(AddressOf TaskWorker, DelayTime, False)
    End Sub

    Private Sub TaskWorker()

        ' Pause Check
        If ActiveTaskMetaData.IsTaskPaused Then
            Me.DestroyComponent()
            Return
        End If

        ' Request Child to populate the queue
        Me.UpdateTransferQueue()

        ' Iterate and process the movement
        For Each Entry In TransferQueue
            ' [Transfer Logic Implementation]
        Next

        ' Restart the loop cycle
        Me.IComponentTask_Restart()
    End Sub


    ' Placeholder for Child overrides.
    Protected Overridable Sub UpdateTransferQueue()
    End Sub

    ' --- Interface Implementations ---
    Public Sub IComponentTask_Restart() Implements BI_ItemSorter_Tasks.IComponentTask_Restart
        KismetSystemLibrary.ClearAndInvalidateTimerHandle(Me, TaskWorkerTimerHandle)
        Me.ScheduleTaskWorker()
    End Sub

    Public Sub IComponentTask_End() Implements BI_ItemSorter_Tasks.IComponentTask_End
        KismetSystemLibrary.ClearAndInvalidateTimerHandle(Me, TaskWorkerTimerHandle)
        Me.DestroyComponent()
    End Sub
End Class
```

</details>

---

### Logistics Optimisation: Inventory-Aware Routing

The `AC_ItemSorter_Task_Base` does not treat all inventories equal. To maximise server performance and eliminate game-thread "hitching," the system utilises a **Switch-on-Inventory-Type** routing architecture. This ensures that "cheap" operations (Dedicated Storage) are executed instantly, while "expensive" operations (Primal Inventories) are time-sliced and throttled.

### 1. The Four Transfer Paths
Distinction is critical because **Dedicated Storage** handles transfers via pure integer maths (no `PrimalItem` object overhead), whereas **Regular Inventories** require expensive array iteration and object reference management.

| Path | Inventory Pairing | Performance Profile | Execution Strategy |
| :--- | :--- | :--- | :--- |
| **D2D** | Dedicated $\rightarrow$ Dedicated | **O(1) - Instant** | Pure integer maths; completes immediately. |
| **D2R** | Dedicated $\rightarrow$ Regular | **Throttled** | Processed in chunks via `ProcessAmountInChunksWithDelay`. |
| **R2D** | Regular $\rightarrow$ Dedicated | **Throttled** | Array iteration via `ForEachLoopInChunksWithDelay`. |
| **R2R** | Regular $\rightarrow$ Regular | **High Cost, Throttled** | Array iteration via `ProcessArrayInChunksWithDelayAndBreak` |

### 2. Latent-Safe Sequencing
ARK: Survival Ascended logic runs on a **single game thread**. A core constraint of Blueprints is that standard loops (For/While) cannot be suspended by `Delay` nodes. To prevent the system from stalling the CPU or causing "Server Snapshots," the `TaskWorker` employs **Latent Non-Blocking Sequencing**:

* **Yielding Execution:** Instead of blocking the thread, the system "yields" control back to the engine. By using the engines `LatentActionManager`, the Sorter pauses its specific logic chain, allows the Game Thread to process vital tasks (like AI and Physics), and resumes in a later frame.

* **Contract:** The `TaskWorker` avoids standard loop nodes. Every transfer path must obey a strict contract: **"The next entry in the TransferQueue must only be triggered by the `Complete` execution pin of the previous Latent action."**

* **Result:** This creates a time-slicing effect where massive resource migrations "breathe" with the server’s framerate rather than executing in a single, frame-killing burst.

### 3. Implementation Logic: Batching & Chunking

To further reduce overhead, the system implements **Item Batching**.

#### **A. Integer-Based Chunking (`ProcessAmountInChunksWithDelay`)**
When pulling from Dedicated Storage, the system moves resources in "Network-Friendly Chunks" (e.g. stacks of 1000). A delay is inserted between chunks to allow the server heartbeat to remain stable.

#### **B. Array-Based Chunking (`ForEachLoopInChunksWithDelay` and `ProcessArrayInChunksWithDelayAndBreak`)**
When iterating through Primal Inventories, the system processes a segment of the item array (e.g. 25 slots) before yielding. This ensures that inventories with hundreds of slots do not cause a CPU spike during the "Inspect" or "Transfer" phases.

---

### 05. The Butler (Maintenance) Subsystem (`AC_ItemSorter_Task_Butler`)
The Butler task is a specialised maintenance worker designed to keep a single specific container stocked with up to 10 different item types.

#### Internal Procedural Logic
1.  **Validation:** Resolves the `DestinationStructure` (stored as a **Soft Object Reference**). If the reference is invalid (structure destroyed), an attempt is made to get the structure used by the stored `StructureID` else the task marked as `NoDestination`.

2. **Delta Calculation:**  
   The system iterates through the `SortingInputs` array on the `PrimalInventoryComponent` of the destination structure.  
   For each entry, it calculates `CurrentCount = GetItemTemplateQuantity(SortingInputs[i])`.

3. **Requirement Calculation:**  
   For each `PrimalInventoryComponent.SortingInputs[i]`, the system searches the `RequestedItems` array (an array of `F_RequestedItemEntry` structs) to find a matching `ItemClass`.  
   - If a match is found, it retrieves the `RequestedQuantity` from that entry.  
   - `QuantityNeeded = RequestedQuantity - CurrentCount`.  
   - If `QuantityNeeded > 0`, the system searches for the required item in the Sorter's cached source list (e.g. Dedicated Storage, Vault, Refrigerator, etc.).  
   - If no matching entry exists in `RequestedItems`, the item is ignored (no action taken for that slot).

4.  **Exact Fill:** It adds a single `F_TransferQueueEntry` for the *exact* amount needed to reach the target, preventing overfilling and wasted movements.

    ```ini
        [ START TASK ]
                |
                v
    +-----------------------+          +-----------------------+
    |  Validate Destination |---(No)-->|  Try Match via        |---(Fail)--> [ NoDestination ]
    |  (Soft Object Ref)    |          |  Stored StructureID   |
    +-----------------------+          +-----------------------+
                | (Success)
                v
    +-----------------------+      +---------------------------+
    | Iterate SortingInputs | <--- | PrimalInventoryComponent  |
    | (Destination)         |      | (Target Structure)        |
    +-----------------------+      +---------------------------+
                |
                v
    +-----------------------+      +---------------------------+
    |  Calculate Delta:     | <--- |   RequestedItems Array    |
    |  ReqQty - CurrentQty  |      | (F_RequestedItemEntry)    |
    +-----------------------+      +---------------------------+
                |
        [ QuantityNeeded > 0? ]---(No)--> [ Ignore Slot ]
                | (Yes)
                v
    +-----------------------+      +---------------------------+
    | Search Cached Sources | <--- | Dedis, Vaults, Fridges,   |
    |    (or World Scan)    |      | etc.                      |
    +-----------------------+      +---------------------------+
                |
                v
    +-----------------------+
    | Exact Fill Transfer   | --> [ Add F_TransferQueueEntry ]
    | (Prevents Overfill)   |
    +-----------------------+
    ```


5. **Saved Butler Tasks & UI Round Trip:**

- The event relays updates the `SavedButlerTasks` `SaveGame` array in a few ways. This keeps the tasks fresh without having a separate timed function updating the information when its not needed.
  - When the `OnWorldScan` Event is triggered in `BP_TekItemSorter` for type `PrimalStructureItemContainers`
  - When `OnCurrentRangeUpdated` Event is triggered from `BP_TekItemSorter`'s parent blueprint.
  - When the UI is requesting tasks via `Buff_ItemSorter`.
  - When a task is modified in the UI and the user saves the modifications. (replicated to server)

- When a Butler task has changed during one of the above update methods, the Butler Task component's stored task data is also refreshed.

- The Butler Task component validates the destination and sources on every call of `RequestUpdateTransferQueue` and updates the `SavedButlerTasks` `SaveGame` entry. These calls measure in the milliseconds and are not that intensive, so worth it to have the data as fresh as possible should players make changes to the structures *"Sorting Inventory"*.
---

<details>
<summary>>Code Structure (pseudo code)</summary>

```vbnet
Public Structure F_StructureData
    Public Property StructureID As Int32
    Public Property PrimalStructure As SoftObjectPtr(Of PrimalStructure)
    Public Property BoxName As String ' Custom structure name set by the player. (only for UI)
    Public Property AssociatedItem As SoftClassPtr(Of PrimalItem) ' Consumes Primal Item class (only for UI)
End Structure

Public Structure F_RequestedItemEntry
    Public Property ItemClass As SoftClassPtr(Of PrimalItem)
    Public Property RequestedQuantity As Integer
    Public Property UsingItemStacks As Boolean
End Structure

' Payload for Butler/Distribution sorting.
Public Structure F_ItemSorterTask_Butler
    Public Property TaskMetaData As F_ItemSorterTask_MetaData
    Public Property DestinationStructure As F_StructureData
    Public Property RequestedItems As Array(Of F_RequestedItemEntry)
    Public Property SourceStructures As Array(Of F_StructureData)
End Structure
```

</details>

---

### Reasoning: Soft Object References
We use **Soft Object References** for actors to optimise replication. Using soft object references for actors in RPCs helps avoid unnecessary hard dependencies on the actor. This reduces the risk of sending invalid or unloaded references, simplifies serialization, and allows the client to resolve the reference lazily without forcing the actor to be loaded immediately. If the structure is mutated on the client, upon return to the server, we can validate or invalidate actor references and prune stale ones or prune the task from the task store.

**Soft class references** are used in the savegame to ensure robustness when mods or assets fail to load. Unlike hard references, soft references store only the class path and do not require the actor or asset to be present. This allows the savegame to retain information about tasks and requested items even if the mod providing the class is missing, avoiding data loss while enabling pruning of invalid object instances.”

#### Key points:
- It’s about safety and lazy resolution, not network bandwidth savings.
- Avoids mandatory actor loading or tight coupling.
- Soft references survive missing mods because they store paths, not instances.
- Hard references would break if the actor/class isn’t loaded, risking corrupted save data.
- Object references can be pruned safely; class references are retained for consistency in tasks like `ItemsToMove`.
- This is especially relevant for custom tasks in mod-heavy server setups.


```md
            ┌─────────────────────────────┐
            │       Actor Reference       │
            └─────────────────────────────┘
                         │
        ┌────────────────┴──────────────────┐
        │                                   │
   Hard (AActor*) Reference           Soft (TSoftObjectPtr<AActor>) Reference
        │                                   │
   Immediate pointer                  Stores path only
   Requires actor loaded              Actor can be unloaded
   Can break RPC if missing           Lazy resolution on client
   Tight coupling                     Safe to prune stale refs
   Might crash if actor destroyed     Lazy resolution on server

───────────────────────────────────────────────────────────────

            ┌─────────────────────────────┐
            │       Class Reference       │
            └─────────────────────────────┘
                         │
        ┌────────────────┴──────────────────┐
        │                                   │
   Hard (UClass*) Reference           Soft (TSoftClassPtr<UObject>) Reference
        │                                   │
   Requires class loaded              Stores class path
   Can break savegame if missing      Robust to missing mods/assets
   Tight coupling                     Enables pruning invalid tasks
   Risk of corrupted data             Savegame retains info safely


```

We use `Soft Object References` for specific actor instances (e.g. `DestinationStructure`) to allow lazy resolution and safe pruning of stale references.
We use `Soft Class References` for item and actor classes (e.g. `ItemsToMove`, `SourceActorClasses`) to ensure savegame robustness when mods are missing or assets fail to load."

---

### 06. The Custom Task Subsystem (`AC_ItemSorter_Task_Custom`)
Used for high-volume, "One-to-Many" or "Many-to-Many" movements (e.g., "Move all Savorous from Dedicated Storage to 10 different Cooking Pots" or "Move All resources types from Inventory A to Dedicated Storage").

#### Internal Procedural Logic
* **Compound Workload:** Upon its scheduled time, it performs a cross-product scan. If 6 item types are selected and 10 destinations are valid, it will generate 60 queue entries. This can be reduced by having tighter filters in the tasks settings such as defined source and destination custom structure name inclusion and or exclusions.
* **Smart "QuantityNeeded" Calculation:** Like the Butler, it checks each destination's current stock against the `MaxAmount` setting to ensure only required items are moved.

---

<details>
<summary>>Code Structure (pseudo code)</summary>

```vbnet

' Payload for Custom/Filter-based sorting.
Public Structure F_ItemSorterTask_Custom
    Public Property TaskMetaData As F_ItemSorterTaskMetaData
    Public Property ItemsToMove As Array(Of SoftClassPtr(Of PrimalItem))
    Public Property MaxAmount As Int32
    Public Property DurabilityPercent As Double
    Public Property DurabilityMath As E_MathFunctions
    Public Property IncludeBlueprints As Boolean = False
    Public Property SourceActorCustomNameContains As String
    Public Property SourceActorCustomNameExcludes As String
    Public Property SourceActorClasses As Array(Of SoftClassPtr(Of Actor))
    Public Property DestinationActorsClasses As Array(Of SoftClassPtr(Of Actor))
    Public Property DestinationActorCustomNameContains As String
    Public Property DestinationActorCustomNameExcludes As String
End Structure
```

</details>

---

### Task Type Comparison: Butler vs Custom

| Feature                  | Butler Task                              | Custom Task                                      |
|--------------------------|------------------------------------------|--------------------------------------------------|
| **Quantity Style**       | Exact fill to target quantity            | Capped fill up to MaxAmount per destination      |
| **Destinations**         | Single fixed structure (e.g. one Tek Gen)| Multiple filtered destinations (Many-to-Many)    |
| **Typical Use Case**     | Maintenance (keep troughs stocked, gens fuelled) | Bulk consolidation, distribution |
| **Overfill Prevention**  | Built-in (never exceeds target)          | Relies on MaxAmount setting                      |
| **Item Selection**       | Up to 10 specific item types with targets| Flexible filters (classes, durability, names)    |
| **Heat Impact**          | Lower (frequent small precise transfers) | Higher (potential for large batch transfers)     |


## 07. The Area Scan Subsystem (`AC_AreaScanManager`) Component.

The `AC_AreaScanManager` is a performance-critical engine of the `BP_TekItemSorter` its attached at compile time. Its primary purpose is to centralise and optimise world-scanning logic, preventing "CPU Spikes" that occur when multiple logistics tasks attempt to scan the world simultaneously.

### Design Intent & Purpose

* **Avoid Redundant Scans:** In a standard logistics mod, 10 tasks might perform 10 separate `SphereOverlapActors` calls every few seconds. The Scan Manager reduces this to one single, lazily triggered call. This ensures that scans are only performed when required, and rapid successive calls use the same cache, while sparse calls update the cache.

* **The "Cache" Philosophy:** Instead of searching the world on demand for every task, tasks "query" the Scan Manager’s internal memory. This memory is updated if the `ScanCacheMaxDurationSeconds` has elapsed (default 20s). This allows the caches to remain relatively fresh without causing CPU bottlenecks by over-scanning the world using expensive overlaps.

* **Distance Ranking:** The manager pre-calculates the distance of all nearby containers, allowing tasks to prioritize "Closest Source" or "Closest Destination" logic without further math overhead. This also keeps the push and pull of items predictable, such as closest served first and, by reverse looping over the cache arrays, furthest consumed first.

---

### Internal Procedural Logic (Lazy Scan Resolution)

The Area Scan Subsystem operates on a lazy evaluation model rather than a fixed “pulse” cycle. Scans are only performed when data is requested and the existing cache is no longer valid.

1.  **On-Demand Scan Triggering:** When a task or UI system requests scan data, the Scan Manager first checks its internal state. If the cached results are still valid (within `ScanCacheMaxDurationSeconds`), no world scan is performed. If the cache is stale or marked dirty, a `SphereOverlapActors` is executed, centred on the `BP_TekItemSorter`.

2.  **Validation & Sorting:** The overlap results are filtered to include only valid `PrimalInventoryComponent` holders. These are then stored in the `AC_AreaScanManager`’s internal arrays, pre-sorted by distance from the sorter.

3.  **Unified Retrieval Interface:** When a `Butler` or `Custom` task needs to locate a container, it calls `IGetStructureContainers` via the `BI_AreaScanManager` interface. When the UI requires an accurate list of structure classes and item classes in range (for example, to populate a “picker” `TileListView` during Custom Task creation), it calls `IGetClassesInRange` via the same interface.

4.  **IInstant Resolution via Cache:** Once populated, the Scan Manager serves all subsequent requests directly from memory until the cache expires or is explicitly invalidated. These cache hits resolve instantly and avoid the CPU cost of repeated world overlap queries.

5. **Overridden Scans:** Overridden Scans: Any call to `BI_AreaScanManager` with `UseCache = False` triggers a fresh scan and updates the cache for that scan type. Similarly, if `OverrideCacheDuration` is provided, it temporarily adjusts the cache validity for that call:

    *Example behaviour:*

    - If `ScanCacheMaxDurationSeconds` = 20 and a request sets `OverrideCacheDuration` = 60, the cache will be used unless more than 60 seconds have elapsed since the last update.

    - If `ScanCacheMaxDurationSeconds` = 60 and a request sets `OverrideCacheDuration` = 10, the cache is considered stale and a fresh scan is performed, updating the cache.

    This mechanism allows tasks to control cache freshness on a per-call basis, ensuring that requests with stricter timing requirements receive up-to-date results, while subsequent tasks that do not override the duration or cache still access the updated information.

---

### Interface Definition (`BI_AreaScanManager`)

The `BI_AreaScanManager` interface defines the contract between the Area Scan Manager and task components, decoupling the tasks from the details of world scanning. Its purpose is to centralize all scanning logic, ensuring tasks never scan the world directly, but instead request information from a single, shared cache maintained by the Scan Manager.

Each function exists to provide specific views of the cached world state for different task requirements:

- `IGetDinoCharacters` – Returns dinos in range filtered by ownership and team. Tasks use this to find targets for logistics.

- `IGetStructureContainers` – Returns all container-type structures in range. Tasks use this to determine sources or destinations for item logistics.

- `IGetStructureContainers_Sortable` – Returns only containers that qualify as sortable (e.g., have input slots or enable sorting toggled). This avoids repeated checks in task logic and respects the cached container list.

- `IGetClassesInRange` – Builds lists of actor and item classes present in the cached containers, dinos, and optionally player inventories. Used by the UI or task configuration systems to populate pickers or class filters without performing scans.

>All functions are designed to respect the cache. Calls can optionally force an update or override the cache duration for specific requests. This ensures tasks with stricter timing or freshness requirements receive up-to-date data without invalidating other cached results.
>
>By enforcing this contract, the interface decouples tasks from scan implementation, guaranteeing that all tasks share a single, synchronized view of the world. This eliminates redundant SphereOverlapActors calls, keeps CPU usage predictable, and ensures consistent, cache-aware results for all tasks.

---

<details>
<summary>Code Structure (pseudo code)</summary>

```vbnet

' Types of Cache
Public Enum E_AreaScanManager_CacheType
    StructureItemContainers
    DinoCharacters
    PlayerCharacters
End Enum

' Types of Cache
Public Enum E_AreaScanManager_Ownership
    SameTeamOnly            ' Tribeless Players or Tribes matching the owning actors TargetingTeam
    OtherTeamOnly           ' Tribeless Players or Tribes NOT matching the owning actors TargetingTeam and NOT Wild or Unclaimed
    WildOnly                ' Only Wild TargetingTeam
    UnclaimedOnly           ' Only Unclaimed TargetingTeam
    NotWildOrUnclaimedOnly  ' Any Tribeless Players or Tribes (Player Owned)
    TribelessOnly           ' TargetingTeam is not a Tribe ID
    TribedOnly              ' TargetingTeam is a Tribe ID
End Enum


' Centralized interface for querying cached area-scan results.
' Implemented by the Area Scan Manager subsystem.
Public Interface BI_AreaScanManager


    ' Gets PrimalDinoCharacter actors in range.
    ' Results are optionally served from cache and filtered by ownership.
    Function IGetDinoCharacters(
        ' Inputs
        ByVal UseCache As Boolean, ' If True, uses cached scan data when valid; otherwise forces a rescan.
        ByVal Ownership As E_AreaScanManager_Ownership, ' Ownership filter applied to the scan results.
        ByVal OtherTeam As Integer, ' Team ID used when Ownership requires an explicit team comparison. Use -1 when not applicable.
        ByVal OverrideCacheDuration As Double ' Overrides the default cache duration in seconds. Use -1 to use the manager default.
        
        ' Outputs
        ByVal DinoCharacters As Array(Of PrimalDinoCharacter) ' An array of PrimalDinoCharacter actors in range.
        ByVal Milliseconds As Int64 ' The time in Milliseconds this took to complete
    )


    ' Gets PrimalStructureItemContainer actors in range.
    ' Results are optionally served from cache and filtered by ownership.
    Function IGetStructureContainers(
        ' Inputs
        ByVal UseCache As Boolean, ' If True, uses cached scan data when valid; otherwise forces a rescan.
        ByVal Ownership As E_AreaScanManager_Ownership, ' Ownership filter applied to the scan results.
        ByVal OtherTeam As Integer, ' Team ID used when Ownership requires an explicit team comparison. Use -1 when not applicable.
        ByVal OverrideCacheDuration As Double ' Overrides the default cache duration in seconds. Use -1 to use the manager default.
        
        ' Outputs
        ByVal StructureContainers As Array(Of PrimalStructureItemContainer) ' An array of PrimalStructureItemContainer actors in range.
        ByVal Milliseconds As Int64 ' The time in Milliseconds this took to complete
    )


    ' Gets structure containers that are valid sorting targets.
    ' A container qualifies if it has sorting inputs or EnableSorting enabled.
    ' (the 10 slots at the bottom of an inventory and the automated withdrawal toggle)
    Function IGetStructureContainers_Sortable(
        ' Inputs
        ByVal UseCache As Boolean, ' If True, uses cached scan data when valid; otherwise forces a rescan.
        ByVal OverrideCacheDuration As Double ' Overrides the default cache duration in seconds. Use -1 to use the manager default.
        
        ' Outputs
        ByVal StructureContainers As Array(Of PrimalStructureItemContainer) ' An array of sortable PrimalStructureItemContainer actors.
        ByVal Milliseconds As Int64 ' The time in Milliseconds this took to complete
    )


    ' Gets classes in range for UI and task configuration purposes.
    ' Includes dino classes, structure classes with inventories,
    ' and optionally PrimalItem classes.
    Function IGetClassesInRange(
        ' Inputs
        ByVal IncludePrimalItems As Boolean, ' If True, includes PrimalItem classes found in dedicated storage.
        
        ' Outputs
        ByVal PrimalItemClasses As Array(Of PrimalItem.Class), ' PrimalItem classes found in range.
        ByVal ActorsWithInventoryClasses As Array(Of Actor.Class), ' Actor classes that contain inventories.
        ByVal DedicatedStorageItemClasses As Array(Of PrimalItem.Class) ' Item classes associated with dedicated storage structures.
        ByVal Milliseconds As Int64 ' The time in Milliseconds this took to complete
    )

End Interface


' Performance-governing component that centralizes world scanning.
' Implements the BI_AreaScanManager interface.
Public Class AC_AreaScanManager
    Inherits ActorComponent
    Implements BI_AreaScanManager

    ' Configuration
    Public Property ScanCacheMaxDurationSeconds As Double = 20.0
    Public Property CurrentRange As Int32 = 10 ' Range in foundations (300 Unreal units per foundation)

    ' Internal Caches
    Private CachedStructures As Array(Of PrimalStructureItemContainer)
    Private CachedStructuresLastTime As Double = 0.0

    Private CachedDinos As Array(Of PrimalDinoCharacter)
    Private CachedDinosLastTime As Double = 0.0

    Private CachedPlayers As Array(Of PrimalCharacter)
    Private CachedPlayersLastTime As Double = 0.0


    ' Returns PrimalDinoCharacter actors in range.
    Public Function IGetDinoCharacters(
        ByVal UseCache As Boolean,
        ByVal Ownership As E_AreaScanManager_Ownership,
        ByVal OtherTeam As Integer,
        ByVal OverrideCacheDuration As Double
    ) Implements BI_AreaScanManager.IGetDinoCharacters

        Dim StartTime As Double = GetWorldTimeSeconds()
        Dim CacheDuration As Double = If(OverrideCacheDuration > 0, OverrideCacheDuration, ScanCacheMaxDurationSeconds)

        ' Check if cache is valid
        If Not UseCache OrElse (GetWorldTimeSeconds() - CachedDinosLastTime) > CacheDuration Then
            UpdateCache(E_AreaScanManager_CacheType.DinoCharacters)
            CachedDinosLastTime = GetWorldTimeSeconds()
        End If

        ' Return Values
        DinoCharacters = CachedDinos.Where(Function(d) FilterByOwnership(d, Ownership, OtherTeam)).ToArray()
        Milliseconds = CLng((GetWorldTimeSeconds() - StartTime) * 1000)

    End Function


    ' Returns all PrimalStructureItemContainer actors in range.
    Public Function IGetStructureContainers(
        ByVal UseCache As Boolean,
        ByVal Ownership As E_AreaScanManager_Ownership,
        ByVal OtherTeam As Integer,
        ByVal OverrideCacheDuration As Double
    ) Implements BI_AreaScanManager.IGetStructureContainers

        Dim StartTime As Double = GetWorldTimeSeconds()
        Dim CacheDuration As Double = If(OverrideCacheDuration > 0, OverrideCacheDuration, ScanCacheMaxDurationSeconds)

        ' Check if cache is valid
        If Not UseCache OrElse (GetWorldTimeSeconds() - CachedStructuresLastTime) > CacheDuration Then
            UpdateCache(E_AreaScanManager_CacheType.Structures)
            CachedStructuresLastTime = GetWorldTimeSeconds()
        End If

        ' Return Values
        StructureContainers = CachedStructures.Where(Function(s) FilterByOwnership(s, Ownership, OtherTeam)).ToArray()
        Milliseconds = CLng((GetWorldTimeSeconds() - StartTime) * 1000)

    End Function


    ' Returns sortable structures by filtering the cached structures on-the-fly.
    Public Function IGetStructureContainers_Sortable(
        ByVal UseCache As Boolean,
        ByVal OverrideCacheDuration As Double,
        ByRef StructureContainers As Array(Of PrimalStructureItemContainer),
        ByRef Milliseconds As Int64
    ) Implements BI_AreaScanManager.IGetStructureContainers_Sortable

        Dim StartTime As Double = GetWorldTimeSeconds()
        Dim CacheDuration As Double = If(OverrideCacheDuration > 0, OverrideCacheDuration, ScanCacheMaxDurationSeconds)

        ' Check if cache is valid
        If Not UseCache OrElse (GetWorldTimeSeconds() - CachedStructuresLastTime) > CacheDuration Then
            UpdateCache(E_AreaScanManager_CacheType.Structures)
            CachedStructuresLastTime = GetWorldTimeSeconds()
        End If

        ' Filter sortable structures on-the-fly
        ' Return Values
        StructureContainers = CachedStructures.Where(Function(s) s.EnableSorting Or s.SortingInputs.length > 0).ToArray()
        Milliseconds = CLng((GetWorldTimeSeconds() - StartTime) * 1000)

    End Function


    ' Returns classes in range for UI and task configuration.
    ' Includes dino classes, structure classes with inventories, 
    ' and optionally PrimalItem classes from those inventories.
    Public Function IGetClassesInRange(
        ByVal IncludePrimalItems As Boolean
    ) Implements BI_AreaScanManager.IGetClassesInRange

        Dim StartTime As Double = GetWorldTimeSeconds()

        ' Local arrays for building results
        Dim LocalPrimalItemsInRange As New Array(Of PrimalItem.Class)
        Dim LocalActorsWithInventoryInRange As New Array(Of Actor.Class)
        Dim LocalDedicatedStorageItemsInRange As New Array(Of PrimalItem.Class)

        ' Get cached structure containers
        Dim LocalStructureContainers = IGetStructureContainers(
            UseCache:=True,
            Ownership:=E_AreaScanManager_Ownership.SameTeamOnly,
            OtherTeam:=-1,
            OverrideCacheDuration:=-1.0
        )

        ' Get cached dino characters
        Dim LocalDinoCharacters = IGetDinoCharacters(
            UseCache:=True,
            Ownership:=E_AreaScanManager_Ownership.SameTeamOnly,
            OtherTeam:=-1,
            OverrideCacheDuration:=-1.0
        )

        ' Process structure containers
        For Each container In LocalStructureContainers
            ' Add the container class to actors with inventories
            LocalActorsWithInventoryClasses.Add(container.GetClass())

            If IncludePrimalItems Then
                ' Add all actual PrimalItems from container inventories
                For Each item In container.MyInventoryComponent.InventoryItems
                    If IsActualPrimalItem(item) Then
                        LocalPrimalItemsInRange.Add(item.GetClass())
                    End If
                Next

                ' Handle dedicated storage structures
                If container.MyInventoryComponent.UsesItemCountInsteadOfInventory Then
                    For Each itemClass In container.MyInventoryComponent.RemoteAddItemOnlyAllowItemClasses
                        LocalDedicatedStorageItemsInRange.Add(itemClass)
                    Next
                End If
            End If
        Next

        ' Process dino characters
        For Each dino In LocalDinoCharacters
            LocalActorsWithInventoryClasses.Add(dino.GetClass())

            If IncludePrimalItems Then
                For Each item In dino.MyInventoryComponent.InventoryItems
                    LocalPrimalItemsInRange.Add(item.GetClass())
                Next
            End If
        Next

        ' Optionally process player inventories
        If IncludePrimalItems Then
            Dim LocalPlayerCharacters = GetPlayerCharacters(
                UseCache:=True,
                OverrideCacheDuration:=-1.0
            )
            For Each player In LocalPlayerCharacters
                For Each item In player.MyInventoryComponent.InventoryItems
                    If IsActualPrimalItem(item) Then
                        LocalPrimalItemsInRange.Add(item.GetClass())
                    End If
                Next
            Next
        End If

        ' Output results
        PrimalItemClasses = LocalPrimalItemsInRange
        ActorsWithInventoryClasses = LocalActorsWithInventoryInRange
        DedicatedStorageItemClasses = LocalDedicatedStorageItemsInRange
        Milliseconds = CLng((GetWorldTimeSeconds() - StartTime) * 1000)

    End Function


End Class
```

</details>

---

### Benefits and Performance Impact

* **Linear CPU Scaling:** Whether a tribe has 1 task or 100 tasks, the "World Scan Cost" remains constant because only one component is performing the search on a timed cache basis.
* **Server Health:** This subsystem is the primary defence against "Server Stutter" during high-volume logistics movements.

---
### 08. UI & List Management
The UI uses a **ListView** and **TileListView** architecture to manage the high volume of tasks or items in item picker windows.

* **Logic:** Only visible widgets are active. Widgets are fed data by `O_ItemSorter_Entry_*` and `O_ItemSelection_Entry` UObjects.
* **Naming Conventions:**
    * **Custom Task Row:** UserWidget:  `UW_LI_ItemSorter_Entry_Custom`
    * **Butler Task Row:** UserWidget: `UW_LI_ItemSorter_Entry_Butler`
    * **Tile Item:** UserWidget: `UW_LI_ItemSelection_Entry`
* **Benefit:** This prevents the UI from causing client-side FPS drops, even if the tribe has dozens of complex maintenance tasks running.

>All internal calculations use absolute Kelvin scale. The player UI displays values in the user's preferred unit (°C / °F / K).

---

### 09. Communication & Safety (Interfaces)
#### **Circular Dependencies** 
To avoid **Circular Dependencies** (a common cause of crashes and "Black Screen" errors in ASA), the system strictly communicates via **Blueprint Interfaces**

### Authoritative Communication

The `BP_TekItemSorter` is a structure and therefore cannot be "owned" by a player's connection. To facilitate **secure, authoritative communication** between the Client UI and the Server-side Sorter, we utilise a **Temporary Player Buff** (`Buff_ItemSorter`).

### Internal Procedural Logic

* **Buff Attachment:** When a player interacts with the Sorter, the structure applies `Buff_ItemSorter` to the player, which opens the user interface `U_ItemSorter_Main`.  

* **Interface Reference:** Upon creation, the Buff is given an **interface reference** to the `BP_TekItemSorter` as `BI_TekItemSorter`. Because the Buff is **owned by the player**, it can successfully execute `Server` and `Client` RPCs.  

* **Command Relay:** When the player clicks "Save," "Delete," or other actions in the UI, the UI calls a function on the Buff. The Buff then executes a **Run on Server RPC**, which in turn calls the corresponding interface function on the Sorter using the stored interface reference.  

* **Clean-up:** Once the UI is closed, the Buff is removed from the player to clear references and stop replication of task data and dispatched events from the structure. All event listeners in the Buff are unbound throughout the widget hierarchy to ensure the garbage collector correctly cleans up unused objects.


---

### 10. Status: Subsystems in Progress
The following areas have been identified but the internal logic/data is currently a **Work in Progress**:
* **Manual Override Penalties:** The exact multiplier applied to heat when a player triggers a task manually via the UI. not started.
* **Risk Level Implementation:** Partially implemented (UI display exists). Full threshold mapping, throttling integration, and hazard effects (damage, fires) are under development.

* **Dynamic Radius Scaling:** Logic to shrink the Scan Manager `CurrentRange` if the `AC_HeatSystem` reports `Critical` heat levels to further save CPU.* **Structure Effects:** Not started; not yet conceptualised.
* **Structure Hazard World Effects:** Not started; not yet conceptualised.
* **Structure Hazard Buff Systems:** Not started; not yet conceptualised.
**Structure Effects:** Not started; not yet conceptualised.

### 11. Passing Ideas that may get implemented.
* **Dynamic Radius Scaling:** Logic to shrink the Scan Manager `CurrentRange` if the `AC_HeatSystem` reports `Critical` heat levels to further save CPU.
* **Priority Filtering:** Logic to allow the Scan Manager to ignore "Ignore-Tagged" structures (e.g., structures with "IGNORE" or "**" in their custom name) or specific structure CustomTag entries.

