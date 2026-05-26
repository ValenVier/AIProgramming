# AI Programming

Third-person game built in Unreal Engine 5.7.4. Two AI guards patrol routes, detect the player through sight and hearing, pursue or investigate, and coordinate with each other.

---

## G Requirements

- NavMesh covers the walkable area
- Two agents patrol autonomously using a Behaviour Tree
- Player-controlled character in the scene
- Sight triggers Chase state, hearing triggers Investigate state
- Agents return to Patrol after losing detection

---

## VG Requirements

- Both senses implemented with distinct responses: sight triggers direct pursuit (ChasingPlayer), hearing triggers location investigation (Investigating)
- Team alertness: when one guard detects the player, other guards in Passive state transition to Investigating via a shared Game Instance flag

---

## Behaviour Tree Structure

Root - Selector
- SequenceChasingPlayerState (highest priority)
  - Decorator: State == ChasingPlayer, aborts self
  - Service: BTService_CheckLostSight - if sight lost, set state to Investigating
  - Tasks: SetMovementSpeed (Sprint) > ClearFocus > MoveTo AttackTarget > FocusTarget > DefaultAttack > Wait
- SequenceInvestigatingState (medium priority)
  - Decorator: State == Investigating, aborts self
  - Tasks: SetMovementSpeed (Jog) > MoveTo PointOfInterest > Wait (5s) > SetStateAsPassive
- SelectorPassiveState (lowest priority)
  - Decorator: State == Passive, aborts self
  - If has patrol route: SetMovementSpeed (Walk) > MoveAlongPatrolRoute > Wait
  - If no patrol route: ClearFocus > Wait (10s)

---

## Blackboard Keys

| Key | Type | Purpose |
|---|---|---|
| State | Enum (E_AIState) | Current state: Passive, ChasingPlayer, Investigating, Frozen, Dead |
| AttackTarget | Object (Actor) | Player actor reference when detected via sight |
| PointOfInterest | Vector | Location to investigate - last known player position or noise source |
| StateKeyName | String | Key name used by the controller to write the State key |
| AttackTargetKeyName | String | Key name used by tasks to read AttackTarget |
| PointOfInterestKeyName | String | Key name used by tasks to read PointOfInterest |

---

## Perception

### AISense_Sight
- Sight Radius: 1500, Lose Sight Radius: 1800, FOV: 60 degrees
- Triggers ChasingPlayer state, writes AttackTarget and PointOfInterest
- BTService_CheckLostSight detects when sight is lost and transitions to Investigating

### AISense_Hearing
- Triggered by Report Noise Event from the player (F key press)
- Provides a location, not an actor reference - triggers Investigating, not Chase
- Agent moves to the noise location; if player is spotted there, sight takes over

### Key difference
Sight gives an actor and triggers direct pursuit. Hearing gives only a location and triggers investigation.

---

## Team Alertness

Implemented via BP_GameInstance with two variables: PlayerDetected (Bool) and LastKnownPlayerLocation (Vector).

1. Guard detects player via sight - writes PlayerDetected = true and LastKnownPlayerLocation to Game Instance
2. BTService_TeamAlert ticks on all guards (0.25s) - if PlayerDetected is true and guard is Passive, transitions to Investigating with LastKnownPlayerLocation
3. Service immediately sets PlayerDetected = false after alerting
4. Each guard returns to Passive independently after BTTask_SetStateAsPassive completes

---

## Patrol System

Each guard has a BP_PatrolRoute spline actor assigned via an instance-editable variable. BTTask_MoveAlongPatrolRoute moves the agent to each spline point and reverses direction at endpoints.

---

## What I Would Add With More Time

- AISense_Damage so guards react to being hit from behind
- Difficulty scaling per guard instance (sight radius, reaction time, patrol speed)
- AI LOD tiers to disable expensive logic for distant guards
- Footstep sounds triggering hearing events based on surface and movement speed