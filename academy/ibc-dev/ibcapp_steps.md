---
title: "Make a Module IBC-Enabled"
order: 7
description: 
tags: 
  - guided-coding
  - dev-ops
  - ibc
---

# Make a Module IBC-Enabled

In this section, you'll build a conceptual SDK blockchain with one module: first as a regular module, and second as an IBC module. This will introduce you to what makes a module IBC-enabled.

## Scaffold a leaderboard chain

By now you should be familiar with scaffolding a chain with Ignite CLI. If not, check out the [Create Your Own Chain](/hands-on-exercise/1-ignite-cli/index.md) chapter.

To begin, scaffold a `leaderboard` chain:

```bash
$ ignite scaffold chain github.com/cosmonaut/leaderboard
```

This creates a chain with `x/leaderboard` a regular SDK module.

Next, scaffold another chain (for example in another git branch), but this time add the `--no-module` flag:

```bash
$ ignite scaffold chain github.com/cosmonaut/leaderboard --no-module
```

Now add the `x/leaderboard` module as an IBC module with the `--ibc` flag:

```bash
$ ignite scaffold module leaderboard --ibc
```

The output you see on the terminal when the module has finished scaffolding already gives a sense of what has to be implemented to create an IBC module:

```bash
modify app/app.go
modify proto/leaderboard/genesis.proto
create proto/leaderboard/packet.proto
modify testutil/keeper/leaderboard.go
modify x/leaderboard/genesis.go
create x/leaderboard/module_ibc.go
create x/leaderboard/types/events_ibc.go
modify x/leaderboard/types/genesis.go
modify x/leaderboard/types/keys.go
```

<HighlightBox type="warning">

The code in this section was scaffolded with Ignite CLI v0.22. This includes ibc-go v3 as a dependency. The latest version of ibc-go is already past v3 so there may be some differences compared to the code in this section. For documentation on the latest version of ibc-go, please refer to the [ibc-go docs](https://ibc.cosmos.network/main/ibc/apps/apps.html).

</HighlightBox>

For a more detailed view, you can now compare both versions with a `git diff`.

<HighlightBox type="tip">

To make use of `git diff`s to check the changes, be sure to commit between different (scaffolding) actions.

```sh
$ git diff <commit_hash_1> <commit_hash_2>
```

You can use git or GitHub to visualize the `git diff`s or alternatively use [diffy.org](https://diffy.org/).

</HighlightBox>

## IBC application module requirements

What does Ignite CLI do behind the scenes when creating an IBC module for us? What do you need to implement if you want to upgrade a regular custom application module to an IBC-enabled module?

The required steps to implement can be found in the [ibc-go docs](https://ibc.cosmos.network/main/ibc/apps/apps.html). There you will find:

<HighlightBox type="info">

**To have your module interact over IBC you must:**

* Implement the `IBCModule` interface:
  * Channel (opening) handshake callbacks
  * Channel closing handshake callbacks
  * Packet callbacks
* Bind to a port(s).
* Add keeper methods.
* Define your packet data and acknowledgment structs as well as how to encode/decode them.
* Add a route to the IBC router.

</HighlightBox>

Now take a look at the `git diff` and see if you can recognize the steps listed above.

### Implementing the `IBCModule` interface

<HighlightBox type="docs">

For a full explanation, visit the [ibc-go docs](https://ibc.cosmos.network/main/ibc/apps/ibcmodule.html).

</HighlightBox>

The Cosmos SDK expects all IBC modules to implement the [`IBCModule` interface](https://github.com/cosmos/ibc-go/tree/main/modules/core/05-port/types/module.go). This interface contains all of the callbacks IBC expects modules to implement. This includes callbacks related to:

* Channel handshake (`OnChanOpenInit`, `OnChanOpenTry`, `OncChanOpenAck`, and `OnChanOpenConfirm`)
* Channel closing (`OnChanCloseInit` and `OnChanCloseConfirm`)
* Packets (`OnRecvPacket`, `OnAcknowledgementPacket`, and `OnTimeoutPacket`).

Ignite CLI implements this in the file `x/leaderboard/module_ibc.go`.

<ExpansionPanel title="x/leaderboard/module_ibc.go">
    
```go
// OnChanOpenInit implements the IBCModule interface
func (am AppModule) OnChanOpenInit(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID string,
    channelID string,
    chanCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version string,
) error {

    // Require portID is the portID module is bound to
    boundPort := am.keeper.GetPort(ctx)
    if boundPort != portID {
        return sdkerrors.Wrapf(porttypes.ErrInvalidPort, "invalid port: %s, expected %s", portID, boundPort)
    }

    if version != types.Version {
        return sdkerrors.Wrapf(types.ErrInvalidVersion, "got %s, expected %s", version, types.Version)
    }

    // Claim channel capability passed back by IBC module
    if err := am.keeper.ClaimCapability(ctx, chanCap, host.ChannelCapabilityPath(portID, channelID)); err != nil {
        return err
    }

    return nil

}

// OnChanOpenTry implements the IBCModule interface
func (am AppModule) OnChanOpenTry(
ctx sdk.Context,
order channeltypes.Order,
connectionHops []string,
portID,
channelID string,
chanCap \*capabilitytypes.Capability,
counterparty channeltypes.Counterparty,
counterpartyVersion string,
) (string, error) {

    // Require portID is the portID module is bound to
    boundPort := am.keeper.GetPort(ctx)
    if boundPort != portID {
        return "", sdkerrors.Wrapf(porttypes.ErrInvalidPort, "invalid port: %s, expected %s", portID, boundPort)
    }

    if counterpartyVersion != types.Version {
        return "", sdkerrors.Wrapf(types.ErrInvalidVersion, "invalid counterparty version: got: %s, expected %s", counterpartyVersion, types.Version)
    }

    // Module may have already claimed capability in OnChanOpenInit in the case of crossing hellos
    // (ie chainA and chainB both call ChanOpenInit before one of them calls ChanOpenTry)
    // If the module can already authenticate the capability then the module already owns it so we don't need to claim
    // Otherwise, the module does not have channel capability and we must claim it from IBC
    if !am.keeper.AuthenticateCapability(ctx, chanCap, host.ChannelCapabilityPath(portID, channelID)) {
        // Only claim channel capability passed back by IBC module if we do not already own it
        if err := am.keeper.ClaimCapability(ctx, chanCap, host.ChannelCapabilityPath(portID, channelID)); err != nil {
            return "", err
        }
    }

    return types.Version, nil

}

// OnChanOpenAck implements the IBCModule interface
func (am AppModule) OnChanOpenAck(
ctx sdk.Context,
portID,
channelID string,
\_,
counterpartyVersion string,
) error {
if counterpartyVersion != types.Version {
return sdkerrors.Wrapf(types.ErrInvalidVersion, "invalid counterparty version: %s, expected %s", counterpartyVersion, types.Version)
}
return nil
}

// OnChanOpenConfirm implements the IBCModule interface
func (am AppModule) OnChanOpenConfirm(
ctx sdk.Context,
portID,
channelID string,
) error {
return nil
}

// OnChanCloseInit implements the IBCModule interface
func (am AppModule) OnChanCloseInit(
ctx sdk.Context,
portID,
channelID string,
) error {
// Disallow user-initiated channel closing for channels
return sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, "user cannot close channel")
}

// OnChanCloseConfirm implements the IBCModule interface
func (am AppModule) OnChanCloseConfirm(
ctx sdk.Context,
portID,
channelID string,
) error {
return nil
}

// OnRecvPacket implements the IBCModule interface
func (am AppModule) OnRecvPacket(
ctx sdk.Context,
modulePacket channeltypes.Packet,
relayer sdk.AccAddress,
) ibcexported.Acknowledgement {
var ack channeltypes.Acknowledgement

    // this line is used by starport scaffolding # oracle/packet/module/recv

    var modulePacketData types.LeaderboardPacketData
    if err := modulePacketData.Unmarshal(modulePacket.GetData()); err != nil {
        return channeltypes.NewErrorAcknowledgement(sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "cannot unmarshal packet data: %s", err.Error()).Error())
    }

    // Dispatch packet
    switch packet := modulePacketData.Packet.(type) {
    // this line is used by starport scaffolding # ibc/packet/module/recv
    default:
        errMsg := fmt.Sprintf("unrecognized %s packet type: %T", types.ModuleName, packet)
        return channeltypes.NewErrorAcknowledgement(errMsg)
    }

    // NOTE: acknowledgment will be written synchronously during IBC handler execution.
    return ack

}

// OnAcknowledgementPacket implements the IBCModule interface
func (am AppModule) OnAcknowledgementPacket(
ctx sdk.Context,
modulePacket channeltypes.Packet,
acknowledgement []byte,
relayer sdk.AccAddress,
) error {
var ack channeltypes.Acknowledgement
if err := types.ModuleCdc.UnmarshalJSON(acknowledgement, &ack); err != nil {
return sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "cannot unmarshal packet acknowledgement: %v", err)
}

    // this line is used by starport scaffolding # oracle/packet/module/ack

    var modulePacketData types.LeaderboardPacketData
    if err := modulePacketData.Unmarshal(modulePacket.GetData()); err != nil {
        return sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "cannot unmarshal packet data: %s", err.Error())
    }

    var eventType string

    // Dispatch packet
    switch packet := modulePacketData.Packet.(type) {
    // this line is used by starport scaffolding # ibc/packet/module/ack
    default:
        errMsg := fmt.Sprintf("unrecognized %s packet type: %T", types.ModuleName, packet)
        return sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, errMsg)
    }

    ctx.EventManager().EmitEvent(
        sdk.NewEvent(
            eventType,
            sdk.NewAttribute(sdk.AttributeKeyModule, types.ModuleName),
            sdk.NewAttribute(types.AttributeKeyAck, fmt.Sprintf("%v", ack)),
        ),
    )

    switch resp := ack.Response.(type) {
    case *channeltypes.Acknowledgement_Result:
        ctx.EventManager().EmitEvent(
            sdk.NewEvent(
                eventType,
                sdk.NewAttribute(types.AttributeKeyAckSuccess, string(resp.Result)),
            ),
        )
    case *channeltypes.Acknowledgement_Error:
        ctx.EventManager().EmitEvent(
            sdk.NewEvent(
                eventType,
                sdk.NewAttribute(types.AttributeKeyAckError, resp.Error),
            ),
        )
    }

    return nil

}

// OnTimeoutPacket implements the IBCModule interface
func (am AppModule) OnTimeoutPacket(
ctx sdk.Context,
modulePacket channeltypes.Packet,
relayer sdk.AccAddress,
) error {
var modulePacketData types.LeaderboardPacketData
if err := modulePacketData.Unmarshal(modulePacket.GetData()); err != nil {
return sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "cannot unmarshal packet data: %s", err.Error())
}

    // Dispatch packet
    switch packet := modulePacketData.Packet.(type) {
    // this line is used by starport scaffolding # ibc/packet/module/timeout
    default:
        errMsg := fmt.Sprintf("unrecognized %s packet type: %T", types.ModuleName, packet)
        return sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, errMsg)
    }

    return nil

}

````

</ExpansionPanel>

Additionally, in the `module.go` file, add the following line (and the corresponding import):

```diff
var (
    _ module.AppModule      = AppModule{}
    _ module.AppModuleBasic = AppModuleBasic{}
    // Add this line
+   _ porttypes.IBCModule   = IBCModule{}
)
```

#### Channel handshake version negotiation

Application modules are expected to verify the versioning used during the channel handshake procedure:

* `OnChanOpenInit` will verify that the relayer-chosen parameters are valid and perform any custom `INIT` logic.
  * It may return an error if the chosen parameters are invalid, in which case the handshake is aborted. If the provided version string is non-empty, `OnChanOpenInit` should return the version string if valid or an error if the provided version is invalid.
  * **If the version string is empty, `OnChanOpenInit` is expected to return a default version string representing the version(s) it supports.** If there is no default version string for the application, it should return an error if the provided version is an empty string.
* `OnChanOpenTry` will verify the relayer-chosen parameters along with the counterparty-chosen version string and perform custom `TRY` logic.
  * If the relayer-chosen parameters are invalid, the callback must return an error to abort the handshake. If the counterparty-chosen version is not compatible with this module's supported versions, the callback must return an error to abort the handshake.
  * If the versions are compatible, the try callback must select the final version string and return it to core IBC.`OnChanOpenTry` may also perform custom initialization logic.
* `OnChanOpenAck` will error if the counterparty selected version string is invalid and abort the handshake. It may also perform custom `ACK` logic.

<HighlightBox type="info">

Versions must be strings but can implement any versioning structure. Often a simple template is used that combines the name of the application and an iteration number, like `leaderboard-1` for the leaderboard IBC module.

However, the version string can also include metadata to indicate attributes of the channel you are supporting, like applicable middleware and the underlying app version. An example of this is the version string for middleware, which is discussed in a [later section](insert-link.com).

</HighlightBox>

#### Packet callbacks

The general application packet flow was discussed in [a previous section](https://tutorials.cosmos.network/academy/4-ibc/channels.html#application-packet-flow). As a refresher, let's take a look at the diagram:

![Packet flow](/academy/ibc-dev/images/packetflow.png)

The packet callbacks in the packet flow can now be identified by investigating the `IBCModule` interface.

##### Sending packets

Modules **do not send packets through callbacks**, since modules initiate the action of sending packets to the IBC module, as opposed to other parts of the packet flow where messages sent to the IBC module must trigger execution on the port-bound module through the use of callbacks. Thus, to send a packet a module simply needs to call `SendPacket` on the `IBCChannelKeeper`.

<HighlightBox type="warning">

In order to prevent modules from sending packets on channels they do not own, IBC expects modules to pass in the correct channel capability for the packet's source channel.

</HighlightBox>

<HighlightBox type="reading">

For advanced readers, more on capabilities can be found in the [ibc-go docs](https://ibc.cosmos.network/main/ibc/overview.html#capabilities) or the [ADR on the Dynamic Capability Store](https://github.com/cosmos/cosmos-sdk/blob/6aaf83c894e917836a047b0399dd70a95fd2710d/docs/architecture/adr-003-dynamic-capability-store.md).

</HighlightBox>

##### Receiving packets

To handle receiving packets, the module must implement the `OnRecvPacket` callback. This gets invoked by the IBC module after the packet has been proved valid and correctly processed by the IBC keepers. Thus, the `OnRecvPacket` callback only needs to worry about making the appropriate state changes given the packet data without worrying about whether the packet is valid or not.

Modules may return to the IBC handler an acknowledgment which implements the `Acknowledgement` interface. The IBC handler will then commit this acknowledgment of the packet so that a relayer may relay the acknowledgment back to the sender module.

The state changes that occurred during this callback will only be written if:

* The acknowledgment was successful as indicated by the `Success()` function of the acknowledgement.
* The acknowledgment returned is nil, indicating that an asynchronous process is occurring.

<HighlightBox type="note">

Applications that process asynchronous acknowledgments must handle reverting state changes when appropriate. Any state changes that occurred during the `OnRecvPacket` callback will be written for asynchronous acknowledgments.

</HighlightBox>
