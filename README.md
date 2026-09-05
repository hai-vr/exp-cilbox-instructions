# temp-cilbox-instructions

The following are instructions for use by an LLM agent to convert a script from the UdonSharp sandbox environment to the Cilbox sandbox environment.

As the LLM agent, you may be tasked to convert a script that was initially designed to run in a sandbox environment called "UdonSharp", into another sandbox environment called "Cilbox".

- The Cilbox sandbox environment uses interpreted C# scripts that only supports a subset of the language features, and a subset of the functions available.
  - As a result, some things possible in the UdonSharp environment might not be possible in the Cilbox environment.
- UdonSharp scripts typically inherit from a class called UdonSharpBehaviour, which does not exist at all in Cilbox and in the codebase you will be working in.
- UdonSharpBehaviour normally provides functions and virtual methods that can be overridden by the implementer.
- Cilbox works differently: The equivalent functions are almost always located in other classes, and virtual methods are replaced by registering some of our methods into listeners located inside other classes.
- You may sometimes find that there is no equivalent in Cilbox for some of the things you may have to convert (such as, some virtual methods have no listener equivalent).
  - In this case you must leave those parts of the code in a non-compilable state and let the user inspect the remaining issues on their own.

## General

- `class Something : UdonSharpBehaviour` generally becomes `[Cilboxable] class Something : MonoBehaviour`
- The Cilbox environment does not allow many of the object accesses. In many cases you may have to look into a selection of classes called "Shims", which provide a sandboxed interface to objects and functions.
  - Use `BasisStringDownloader` to perform URL requests for data such as JSON files.
  - Use `BasisJson.Parse` to parse JSON.
  - You cannot use `MonoBehaviour.Invoke` to delay a call.

## Sandbox quirks

- Never use `Array.Empty<...>()`. Using it may cause the following errors:
  - `Privilege failed for System.Array.Empty generic argument 0 type`
  - `CilboxException: Error: Could not find reference to: [mscorlib][System.Array]`
- Trying to write a bool into a bool array fails with the following error:
  - `not a widening conversion`
  - Recommended fix for now is to use a byte array instead, but add inline comments to signal that the data was originally a bool.

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
 
- `_network.IsLocalOwner()` and `_network.SendCustomNetworkEvent(...)` cannot normally be used on `Start()` (and sometimes `OnEnable()`) as it's too early. Network operation can only occur after `NetworkReady` callback has been called once.
  - If this pattern exists, then delegate it to execute on `WhenNetworkReady`.
 
## Synced variables

- To network anything, use `_network.NetworkReady += WhenNetworkReady` and `_network.NetworkMessageReceived += WhenNetworkMessageReceived`
  - `NetworkReady` is called when we're ready to transmit data.
  - When we are ready, we should ask the network owner to initialize us by sending them a packet.
  - The network owner should listen to that packet, and then send us the initial data using another packet.
    - The network owner should send that packet specifically to us; not to everyone.
  - When we receive data, we should verify that the data actually comes from the network owner using `_network.CurrentNetworkId`.
- If `OnDeserialization` exists, we should execute `OnDeserialization` after the packet has been read.
- If `OnSerialization` exists, we should execute `OnSerialization` before the packet is sent.
- If `RequestSerialization` is called, we should prepare to send a packet at the end of that frame.
- Replace `[UdonSynced]` with `/*[UdonSynced]*/`, and implement transmit those variables as part of our object state in the packet.


## Encoding data

- You cannot use `using` patterns such as `using (MemoryStream ...` nor `using (BinaryReader ...` nor `using (BinaryWriter ...`
- You should use `BitConverter` as needed.
- If you need to encode `Quaternions`, you must use `BasisCompression.QuaternionCompressor.CompressQuaternion` and `BasisCompression.QuaternionCompressor.DecompressQuaternion`
 
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
