# temp-cilbox-instructions

## General

- `class Something : UdonSharpBehaviour` generally becomes `[Cilboxable] class Something : MonoBehaviour`
- Never use `Array.Empty<...>()`. Using it may cause the following errors:
  - `Privilege failed for System.Array.Empty generic argument 0 type`
  - `CilboxException: Error: Could not find reference to: [mscorlib][System.Array]`

## Objects

- Replace `VRCObjectSync` with `BasisPickupSyncNetworking`
- If a class has `Interact()` implemented, add a `BasisInteractableButton`
  - Register an event listener, such as `interactable.OnInteractStartEvent.AddListener(WhenInteractStart)`;
- Replace `VRCPickup` with `BasisPickupInteractable`
  - Register an event listener, such as `pickup.OnInteractStartEvent.AddListener(OnPickup)` and `pickup.OnInteractEndEvent.AddListener(OnDrop)`
  - For the trigger events, use `pickup.OnPickupUse.AddListener(OnPickupUse`). The callback method has an enum value that tells you when the trigger press starts, continues being held, or ends.
- UI Buttons:
  - To trigger events when a UI button is pressed, use `_button.onClick.AddListener(WhenButtonClicked);`
  - UI Buttons should not call `SendCustomEvent` targeting our object.

## Network

- If we need any of the below functions stemming from `_network`, then initialize it in `Start()` using `SafeUtil.MakeNetworkable(this)`
- If `VRCPlayerApi.IsOwner(obj)` is used, and `obj` refers to our own object, then use `_network.IsLocalOwner()`
- If we need `public override void OnPlayerJoined(VRCPlayerApi player)`, then:
  - Replace with `private void WhenPlayerJoined(BasisNetworkPlayer player)`
  - Add `_network.PlayerJoined += WhenPlayerJoined;` to `Start()`
- If we need `public override void OnPlayerLeft(VRCPlayerApi player)`, then:
  - Replace with `private void WhenPlayerLeft(BasisNetworkPlayer player)`
  - Add `_network.PlayerLeft += WhenPlayerLeft;` to `Start()`
 
## Synced variables

- To network anything, use `_network.NetworkReady += WhenNetworkReady` and `_network.NetworkMessageReceived += WhenNetworkMessageReceived`
  - `NetworkReady` is called when we're ready to transmit data.
  - When we are ready, we should ask the network owner to initialize us by sending them a packet.
  - The network owner should listen to that packet, and then send us the initial data using another packet.
- If `OnDeserialization` exists, we should execute `OnDeserialization` after the packet has been read.
- If `OnSerialization` exists, we should execute `OnSerialization` before the packet is sent.
- If `RequestSerialization` is called, we should prepare to send a packet at the end of that frame.
- Replace `[UdonSynced]` with `/*[UdonSynced]*/`, and implement transmit those variables as part of our object state in the packet.

## Encoding data

- You cannot use `using` patterns such as `using (MemoryStream ...` nor `using (BinaryReader ...` nor `using (BinaryWriter ...`
- You should use `BitConverter` as needed.
 
## OnEnable pattern

If `OnEnable` is required, then it must be modified to follow this pattern:

```csharp
        private void Start() { WhenEnabled(); }
        private void OnEnable() { WhenEnabled(); }

        private void WhenEnabled()
        {
            if (_isEnabled) return;

            vrware.SetText(BasisVRWare.MsgMinigameJump);
            _prevPos = BasisPlayersShim.Local.GetPosition();
            _needsEval = true;

            _isEnabled = true;
        }

        private void OnDisable()
        {
            _isEnabled = false;
        }
```

This is because `OnEnable` is emulated incorrectly, so it may not execute properly the first time.

## Teleport

- Teleportation should be done using `BasisLocalPlayer.Instance.Teleport(some.position, some.rotation)`
  - Replace `.TeleportTo(...)` with it.
