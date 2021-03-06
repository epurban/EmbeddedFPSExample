# Room System part 1 (Server-side)

We will create a very basic room system for our FPS game. It will work like this:
- Players can join and leave game rooms.
- Players can create a room with a name and an amount slots.( not in the tutorial but you can add it :smiley:)

Since 2018.3 Unity can ghandle multiple physics scenes which makes it much easier to run multiple rooms on a single server. Before 2018.3 this option didn't exist. One option to do it was to just move each room far away from the others by spawning each room at an icrementing offset but that had some performance disbenefits.

Our Room system will spawn a map and objects for each room and then create a separate physics scene for that room.

Let's start by creating a "Room" script in the Scripts folder and filling it with:

```csharp
using System.Collections.Generic;
using DarkRift;
using UnityEngine;

public class Room : MonoBehaviour
{
    [Header("Public Fields")]
    public string Name;
    public List<ClientConnection> ClientConnections = new List<ClientConnection>();
    public byte MaxSlots;
	
    private Scene scene;
    private PhysicsScene physicsScene;

    public void Initialize(string name, byte maxslots)
    {
        Name = name;
        MaxSlots = maxslots;

        CreateSceneParameters csp = new CreateSceneParameters(LocalPhysicsMode.Physics2D);
        scene = SceneManager.CreateScene("Room_" + name, csp);
        physicsScene = scene.GetPhysicsScene();

        SceneManager.MoveGameObjectToScene(gameObject, scene);
    }
}
```
At the moment this is just a class to hold ClientConnections. In addition we create a new scene in the Inititialize() function and move the gameobject to which the Room is attached to into that new scene. Finally we cache the physicsScene for later use.

We will also need a Manager to manage all rooms. So create a "RoomManager" script in the scripts folder.
Our RoomManager will look like this:
```csharp
using System.Collections.Generic;
using DarkRift;
using DarkRift.Server;
using UnityEngine;

public class RoomManager : MonoBehaviour
{

    public static RoomManager Instance;

    [Header("Prefabs")]
    public GameObject RoomPrefab;

    Dictionary<string, Room> rooms = new Dictionary<string, Room>();

    void Awake()
    {
        Instance = this;
        CreateRoom("Main",25);
        CreateRoom("Main 2", 15);
    }

    public void CreateRoom(string name, byte maxslots)
    {
        GameObject go = Instantiate(RoomPrefab);
        Room r = go.GetComponent<Room>();
        r.Initialize(name, maxslots);
        rooms.Add(name, r);
    }

    public void RemoveRoom(string name)
    {
        rooms.Remove(name); 
    }
}
```
In this script we have a reference to a room prefab which we instantiate to spawn a new room and a dictionary to find rooms by name.

The CreateRoom() function creates a room. It instantiates a copy of the prefab. It initializes the room component that we created earlier with a name and a maxSlot value and adds the room to the dictionary.

The RemoveRoom() function removes a room from the list but does not delete it yet we will do that later.

Now create a RoomManager gameobject in the Main scene of the server and ad the RoomManager component to it.

As a next step we create the Room Prefab:
- Create a new GameObject call it "Room".
- Add a Room component to the GameObject.
- Add a plane as a child to the room.
- scale the plane up to (5,5,5).
- Drag the Room into the prefabs folder.
- Delete the Room from the Scene.
- Drag the Room in the RoomPrefab field of the RoomManager.

Now the basics for our room system are done. We can now create a data object to send information to the player.
to do that open the Networking Data script **again on the client because it's in the shared folder**

Create a RoomData object:
```csharp
public struct RoomData : IDarkRiftSerializable
{
    public string Name;
    public byte Slots;
    public byte MaxSlots;

    public RoomData(string name, byte slots, byte maxSlots)
    {
        Name = name;
        Slots = slots;
        MaxSlots = maxSlots;
    }

    public void Deserialize(DeserializeEvent e)
    {
        Name = e.Reader.ReadString();
        Slots = e.Reader.ReadByte();
        MaxSlots = e.Reader.ReadByte();
    }

    public void Serialize(SerializeEvent e)
    {
        e.Writer.Write(Name);
        e.Writer.Write(Slots);
        e.Writer.Write(MaxSlots);
    }
}
```

Players will receive one of these for each room. It contains its name and the number of slots available and used.

Now change the lobby info the include RoomDatas:

```csharp
public struct LobbyInfoData : IDarkRiftSerializable
{
    public RoomData[] Rooms;

    public LobbyInfoData(RoomData[] rooms)
    {
        Rooms = rooms;
    }

    public void Deserialize(DeserializeEvent e)
    {
        Rooms = e.Reader.ReadSerializables<RoomData>();
    }

    public void Serialize(SerializeEvent e)
    {
        e.Writer.Write(Rooms);
    }
}
```

Now we just need a way to fetch the Roomdata[] array. We will do this in the RoomManager class.
Add the following function to the RoomManager:

```csharp
    public RoomData[] GetRoomDataList()
    {
        RoomData[] datas = new RoomData[rooms.Count];
        int i = 0;
        foreach (KeyValuePair<string, Room> kvp in rooms)
        {
            Room r = kvp.Value;
            datas[i] = new RoomData(r.Name, (byte)r.ClientConnections.Count, r.MaxSlots);
            i++;
        }
        return datas;
    }
```

::: tip 
You'd cache that in a real application.
:::

Finally we have to edit a line in ClientConnection because we changed the constructor of LobbyInfoData.
from:
```csharp
    using (Message m = Message.Create((ushort)Tags.LoginRequestAccepted, new LoginInfoData(client.ID, new LobbyInfoData())))
        {
            client.SendMessage(m, SendMode.Reliable);
        }
```
to:
```csharp
    using (Message m = Message.Create((ushort)Tags.LoginRequestAccepted, new LoginInfoData(client.ID, new LobbyInfoData(RoomManager.Instance.GetRoomDataList()))))
        {
            client.SendMessage(m, SendMode.Reliable);
        }
```

Your scripts should look like this:
- [Room](https://pastebin.com/znpzuPX4)
- [RoomManager](https://pastebin.com/eUBeMCYz)
- [Networking Data](https://pastebin.com/USTdSuLK)
- [ClientConnection](https://pastebin.com/FCH3UCyu)

To finish the room system we still have to add function to join rooms but we do that later first we will implement the client side.