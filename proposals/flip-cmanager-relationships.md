# What we currently do in `master`

## Constructing water

This is what `CClientWater`'s constructors look like:

```cpp
// CClientWater.h
CClientWater(CClientWaterManager* pWaterManager, ElementID ID, CVector& vecBL, CVector& vecBR, CVector& vecTL, CVector& vecTR, bool bShallow = false);
CClientWater(CClientWaterManager* pWaterManager, ElementID ID, CVector& vecL,  CVector& vecR,  CVector& vecTB,                 bool bShallow = false);
```

And in `CPacketHandler::Packet_EntityAdd`, under `WATER`, we have something like this:

```cpp
// CPacketHandler.cpp
if (ucNumVertices == 3)
    pWater = new CClientWater(pWaterManager, EntityID, vecVertices[0], vecVertices[1], vecVertices[2], bShallow);
else
    pWater = new CClientWater(pWaterManager, EntityID, vecVertices[0], vecVertices[1], vecVertices[2], vecVertices[3], bShallow);
```

## Adding and removing from lists

Inside `CClientWater`'s constructor, we have this code:

```cpp
// CClientWater.cpp
CClientWater(...) {
    // Do some assignments
    // [...]

    m_pWaterManager->AddToList(this);
}
```

And inside `CClientWater`'s destructor, we have this code:

```cpp
// CClientWater.cpp
~CClientWater() {
    Unlink();
    Destroy();
}

void Unlink() {
    m_pWaterManager->RemoveFromList(this);
}

// Destroy contains code that tells game_sa (CWaterManagerSA) to destroy the CWaterPoly instance
// This proposal is only for the deathmatch module, so we don't care about changing inside the destructuor.
void Destroy();
```

# I think we should flip the relationship.

Instead of telling a new water instance to associate itself with a manager, we should let the manager birth a new water instance.

The aim would be to facilitate something like this:

```cpp
// CPacketHandler.cpp
if (ucNumVertices == 3)
    pWater = pWaterManager->New(EntityID, vecVertices[0], vecVertices[1], vecVertices[2], bShallow);
else
    pWater = pWaterManager->New(EntityID, vecVertices[0], vecVertices[1], vecVertices[2], vecVertices[3], bShallow);
```

# Problem

**We want to avoid duplicating the constructor**

One way we could implement `CClientWaterManager::New` is by copy-pasting the constructor signature, like so:

```cpp
// CClientWaterManager.h
CClientWater* CClientWaterManager::New(ElementID ID, CVector& vecBL, CVector& vecBR, CVector& vecTL, CVector& vecTR, bool bShallow = false);

// CClientWater.h
CClientWater::CClientWater(ElementID ID, CVector& vecBL, CVector& vecBR, CVector& vecTL, CVector& vecTR, bool bShallow = false);
```
Also, we would make `CClientWater`'s constructor private, and just add `CClientWaterManager` as a friend class. This would prevent anyone from creating random `CClientWater` instances.

I think this is bad, so we should use templating magic to reduce copy-pasting:
```cpp
// CClientWaterManager.h
template<typename... Args>
CClientWater* CClientWaterManager::New(Args&& args...)
{
    // Do some assignments
    // [...]
    /* some variable */ = new CClientWater(std::forward<Args>(args...));
}
```
