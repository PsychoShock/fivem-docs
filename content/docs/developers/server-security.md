---
title: Make your server more secure
draft: true
---

Our anti-cheat cannot protect some of the exploit your server might have. In this simple guide, we'll try to show you how to get started with some tips to prevent in Lua. 

## Events
Some cheats authorize the client to trigger events allowing them to have access to a wide possibilities, let's see how you can protect them:

### Proper Use of Event Handlers in Lua
When working with events in Lua, it's crucial to register them correctly based on whether they are called by the client or the server. A common mistake is registering server events that are not supposed to be called by the client, or vice-versa, which can lead to security vulnerabilities. Here’s a concise guide to using `AddEventHandler` and `RegisterNetEvent` appropriately:

#### `AddEventHandler`

Use `AddEventHandler` when the event is intended to be triggered within the same context, either client-client or server-server. This ensures that the events are not networked and cannot be called by the opposing side.

```lua
AddEventHandler("eventName", function(eventParam1, eventParam2)
    -- Code here will be executed once the event is triggered within the same context.
end)
```

#### `RegisterNetEvent`

Use `RegisterNetEvent` when the event needs to be triggered across different contexts, such as from client to server or server to client.

```lua
RegisterNetEvent("eventName", function(eventParam1, eventParam2)
    -- Code here will be executed once the event is triggered across different contexts.
end)
```

You can learn more [on this guide](docs/scripting-manual/working-with-events/listening-for-events/).

### Add Statement Checks
Even if you build a strong anti-cheat, adding statement checks on **server** events makes them more secure. This is a good practice, although it doesn't prevent everything. Below, we will share some good tips.

When using networked events, make sure to add some checks like:
- Player money
- Player [state bags](/docs/scripting-manual/networking/state-bags/)
- Player inventory items
- Player [position](https://docs.fivem.net/natives/?_0x1647F1CB)
- Player experience and level
- Player [permissions](https://forum.cfx.re/t/basic-aces-principals-overview-guide/90917) and roles

Make sure to retrieve all values using server-side methods, not allowing players to change the values. This ensures the integrity and security of your game environment. Please note that client checks can also be a good pratice but can be easily be ovverided.

## SQL
Let's take an example from a popular [SQL resource](https://github.com/overextended/oxmysql):

### Comparison of SQL Queries

#### Example 1
```lua
local row = MySQL.single('SELECT `firstname`, `lastname` FROM `users` WHERE `identifier` = ?', {
    identifier
})
```

#### Example 2
```lua
local action = 'SELECT'
local database = 'users'
local search = 'WHERE `identifier` = '..identifier..''
local row = MySQL.single(action..' `firstname`, `lastname` FROM'..database..search, {})
```

### Differences and Analysis

#### Example 1
- **Parameter Binding**: Uses parameter binding with a placeholder `?` for the `identifier`.
- **Security**: This approach is more secure because it prevents SQL injection attacks by escaping the input values properly.
- **Readability**: The query is concise and easy to read.
- **Maintenance**: Easier to maintain and modify because the query string is static and any changes are straightforward.

#### Example 2
- **String Concatenation**: Constructs the SQL query using string concatenation with variables `action`, `database`, and `search`.
- **Security**: This approach is insecure and vulnerable to SQL injection attacks. If `identifier` contains malicious code, it will be directly included in the query.
- **Readability**: The query is constructed dynamically, making it harder to read and understand at a glance.
- **Maintenance**: More difficult to maintain due to the dynamic construction of the query string.

### Potential Consequences of SQL Injection

- **Unauthorized Access**: Attackers can bypass authentication mechanisms and gain unauthorized access to user data.
- **Data Manipulation**: Attackers can modify, delete, or insert data, leading to data corruption or loss.
- **System Compromise**: If the database contains sensitive information or controls critical parts of the application, a successful SQL injection can lead to a complete system compromise.

To mitigate the risks of SQL injection and ensure the security, it is essential to use parameterized queries [(as shown in Example 1)](#Example1). This practice effectively prevents the injection of malicious SQL code and protects the database from unauthorized access and manipulation.

## Cheaters
Even thought we are trying to keep our community safe, some users might have not been detected yet. This is some tools to prevents some popular actions:

Please note that this doesn't mean it will block the features presented, but can prevent some cases.

### Teleporation/NoClip
This is a popular function allowing the cheater to move around the map easily which allow to do some actions in the server easier.

```lua
local wasFastTravelAllowed = false
CreateThread(function()
    local lastDist = GetEntityCoords(PlayerPedId())
    local admin = IsAceAllowed('noclip') -- can be replace by any ace you wish
    if admin then return end -- will check if player is admin and not run the thread
    while true do
        local player = PlayerPedId()
        local playerCoords = GetEntityCoords(player)
        local dist = #(playerCoords - lastDist)
        if dist >= 100.0 then
            if not wasFastTravelAllowed then
                SetEntityCoords(player, playerCoords.x, playerCoords.y, playerCoords.z, true, false, false)
                print("Reverting player position -- fast travel not allowed")
                -- Trigger any event to log
            end
        end
        lastDist = playerCoords

        Wait(500)
    end
end)

local playerFastTravel = function()
    while IsPlayerTeleportActive() do wasFastTravelAllowed = true Wait(1) end
    wasFastTravelAllowed = false
end
exports("allowFastTravel", playerFastTravel);
```

Let's break some of the code in parts:
```lua
local wasFastTravelAllowed
```
So `wasFastTravelAllowed` will be crucial here since it will determinate if player had access to teleportation or not. This is checked after the "teleportation" is detected if player was really doing this action or not. Please refer to the other parts to know how to activate this variable.

```lua
allowFastTravel
```
This export is the most important wich will need to be set before teleportation by the player allowing to do this action. You can rename this export as you wish.

```lua
IsPlayerTeleportActive()
```
While setting the export in each teleportation player can do, you also need to make sure using [StartPlayerTeleport](https://docs.fivem.net/natives/?_0xAD15F075A4DA0FDE) and [StopPlayerTeleport](https://docs.fivem.net/natives/?_0xC449EDED9D73009C) since the function allowing the player to teleport with `wasFastTravelAllowed` check if player is using this function. Not using those natives might cause some false positives reports.

Note that this not a copy/paste code, this just explain how to prevent some cases where cheaters might be caught.

### Screenshot
Taking a screenshot of the player's screen can be an effective method to monitor unusual activities that might indicate cheating. You can use [screenshot-basic](https://github.com/citizenfx/screenshot-basic) to capture what they are doing.


### General report function
You can create some usefull functions to generalize every report in one, here's an example:
```lua
local reportsDone = {}
function reportCheater(enumReason, requestScreenshot, preventSpam)
	if reportsDone[enumReason] then return end

	if preventSpam then
		reportsDone[enumReason] = true

        Citizen.SetTimeout(60 * 1000 * 10, function() -- allow 10 min between each report
			reportsDone[enumReason] = false
		end)
	end

    local screenshot
    if requestScreenshot then
        exports['screenshot-basic']:requestScreenshot(function(data)
            screenshot = data
        end)
    end

	-- trigger server event
end
```

**enumReason:** Can be use te enumerize a list of reason this function is trigger.
**requestScreenshot:** Can be use to request a screenshot for the player on warn.
**preventSpam:** Can be use to prevent some things that might pass `reportCheater` function more than 1 time. This will block the specific alert for 1 every 10 minutes.

## Server owner options
#### Please note that the following shouldn't be touch unless you know what you are doing.
Adhesive team is always work really hard to prevent cheaters to be able to use them. You can have most of those features enabled by default with `8450` Fx build version and higher.
<!-- Reference https://discord.com/channels/779705925577080842/842554061734936596/1197336243038597263 -->

```
sv_kick_players_cnl_timeout_sec
```
This is the timeout at which the server will kick the player. (EX: if this is 600, kick them after 10 minutes of no CnL connection).

```
sv_kick_players_cnl_update_rate_sec
``` 
This is the frequency at which CnL is queried with the playerlist.

```
sv_pure_verify_client_settings
``` 
Replaces the periodic request to `info.json` in the client. Establishes a connection between `adhesive`<->`svadhesive` and verifies some of the `sv_settings` such as pureLevel, scripthook and other settings.

```
sv_kick_players_cnl_consecutive_failures
```
How many X's in a row do we need to see a player over the `timeout_sec` in order to Kick. The default is set to 2, indicating that if a player fails to check in for 10 minutes and then misses the next check-in update, they will be kicked. This serves as a failsafe mechanism.

```
sv_authMaxVariance
```
**Variance** is how likely the user's id will change for a given provider (i.e. 'steam', 'ip', or 'license'). You can learn about it [here](docs/server-manual/erver-commands/#sv_authmaxvariance-newvalue).

```
sv_authMinTrust
```
**Trust** is how unlikely it is for the user's identity to be spoofed by a malicious client. You can learn about it [here](docs/server-manual/erver-commands/#sv_authmintrust-newvalue).

```
sv_filterRequestControl
```
A console variable used to block `REQUEST_CONTROL_EVENT` routing based on a configurable policy. You can learn about the list [here](docs/server-manual/erver-commands/#sv_filterrequestcontrol-mode).

### Results on Player
Having those convars activate will likely get the player kick with the following reason:
```
Connection to CNL timed out.
```

**Note:** some error *might occur* but it's unlikely. 

User getting kick doesn't mean is automatically Global Ban. However, it provides an indication of player reliability, which can be useful for assessing trustworthiness.

We also have some other options:

```
sv_disableClientReplays
```
Enabling this will aim to reduce chances of cheating options. Please note that this will disable Rockstar Editor.

## Important to know
The codes provided are not suppose to be working on a copy/paste method. This is just some tips to prevent some actions that might happen in the server. This require some knowledge, you are always free to join our [discord](discord.gg/fivem) to get additionnal help.
