(원문) https://miragenet.github.io/Mirage/

# 설치 [🔗](https://miragenet.github.io/Mirage/docs/general/getting-started#installation)

- OpenUPM 추가
- Mirage 패키지 설치

# 네트워크 매니저

다음에 대한 빠른 접근을 제공하는 매니저 클래스 입니다.

- 네트워크 서버 및 클라이언트
- 네트워크 씬 매니저
- 서버 및 클라이언트 오브젝트 관리자

# RPC (원격 프로시저 호출)

- 컴퓨터 프로그램이 다른 주소 공간에서 원격 제어를 위한 프로그래머의 코딩 없이 함수나 프로시저의 실행을 허용하는 기술
- RPC를 사용하면 프로그래머 함수가 실행 프로그램의 로컬 위치에 있든, 원격 위치에 있든 동일한 코드를 이용하여 함수를 호출할 수 있습니다.
- 이는 개체 지향 소프트웨어에서도 사용되며, 원격 호출이라고도 합니다.
- RPC를 통해 다른 컴퓨터 프로세스의 함수를 호출하거나 데이터를 주고받을 수 있습니다.
- RPC는 원격 호출에 사용되는 프로토콜이지만 세부적인 네트워크 전송 시에는 하부 프로토콜 중 하나를 사용하도록 설계되었습니다. (TPC/IP, IPX, Name Pipe 등)
- Mirage RPC에는 Client RPC, Server RPC, Network Messages 유형이 있습니다.

# Client RPC

- 서버의 NetworkBehaviour에서 클라이언트의 Behaviour로 전송되는 RPC
- 함수를 Client RPC로 만들려면 함수 위에 [ClientRpc] 를 추가하면 됩니다.
- Client RPC 함수는 static으로 선언할 수 없으며 void를 반환해야 합니다.

## RpcTarget에 따른 동작

**RpcTarget.Observers (기본 값)**

- 해당 오브젝트의 Network Visibility에 따라 오직 관찰자들(Observers)에게만 RPC 메시지를 보냅니다.
- 오브젝트에 Network Visibility가 없을 경우 모든 플레이어에게 보내집니다.

**RpcTarget.Owner**

- 해당 오브젝트의 소유자에게만 메시지를 보냅니다.

**RpcTarget.Player**

- 호출 시 전달된 NetworkPlayer에게만 RPC 메시지를 보냅니다.

```csharp
[ClientRpc(target = RpcTarget.Player)]
public void MyRpcFunction(NetworkPlayer target) 
{
    // Code to invoke on client
}
```

- 메시지를 어디로 보낼지 결정하기 위해 인자로 **NetworkPlayer target**을 사용하지만, 실제로 대상 값을 보내지는 않기 때문에 클라이언트에서 target의 값은 항상 null 입니다.

# Exclude owner

- 소유자를 제외한 클라이언트에게 Client RPC 메시지를 보내고 싶을 때 사용하는 옵션으로, 다음과 같이 설정할 경우 Owner를 제외한 모든 클라이언트에게 메시지가 전송됩니다.

```csharp
[ClientRpc(excludeOwner = true)]
```

# Channel

RPC는 신뢰할 수 있는 채널과 신뢰할 수 없는 채널을 사용하여 메시지를 전송할 수 있습니다.

- Channel.Reliable - 신뢰할 수 있는 채널
- Channel.Unrreliable - 신뢰할 수 없는 채널

```csharp
[ClientRpc(channel = Channel.Reliable)]
```

## 값의 반환

- Client RPC는 RpcTarget이 Player 또는 Owner인 경우에만 값을 반환할 수 있습니다.
- 클라이언트가 응답하는 데 시간이 오래 걸릴 수 있으므로 반환 값은 서버에서 기다릴 수 있는 UniTask 여야만 합니다.
- 값을 반환하려면 UniTask<리턴 타입> 과 같은 형태로 사용하면 되며, 리턴 타입으로는 Mirage에서 지원하는 타입들을 지정할 수 있습니다.
- 클라이언트에서는 메서드를 async로 만들거나 다음과 같이 UniTask.FromResult(myResult) 를 사용할 수 있습니다.

```csharp
[ClientRpc]
public UniTask<int> CalculateSumAsync(int a, int b)
{
    int sum = a + b;
    return UniTask.FromResult(sum);
}
```

# Example

```csharp
public class Player : NetworkBehaviour
{
    private int health;

    public void TakeDamage(int amount)
    {
				// 만약 이 코드가 서버에서 실행되지 않았다면 함수 실행을 중
        if (!IsServer)
        {
            return;
        }

        health -= amount;
        Damage(amount);
    }

    [ClientRpc]
    private void Damage(int amount)
    {
        Debug.Log("Took damage:" + amount);
    }
}
```

- Host로 게임을 실행하면 로컬 클라이언트에서도 ClientRpc 호출이 발생합니다.
- 이는 서버와 동일한 프로세스에 있더라도 로컬 클라이언트에서 호출이 된다는 의미로, 따라서 로컬 및 원격 클라이언트의 ClientRpc 호출 동작은 동일합니다.
- target 매개변수를 사용하여 어떤 클라이언트가 호출되는지 지정할 수 있습니다.

- 오브젝트를 소유한 클라이언트만 호출하려면  [ClientRpc(target = RpcTarget.Owner)]를 사용하거나 [ClientRpc(target = RpcTarget.Player)]를 사용하여 메시지를 받을 클라이언트를 지정하고 플레이어를 매개변수로 전달할 수 있습니다.  (아래 예시 참고)

```csharp
public class Player : NetworkBehaviour
{
    private int health;

		// 서버에서만 실행되는 함수
    [Server]
    private void Magic(GameObject target, int damage)
    {
        target.GetComponent<Player>().health -= damage;

        NetworkIdentity opponentIdentity = target.GetComponent<NetworkIdentity>();
        DoMagic(opponentIdentity.Owner, damage);
    }

		
    [ClientRpc(target = RpcTarget.Player)]
    public void DoMagic(INetworkPlayer target, int damage)
    {
        // This will appear on the opponent's client, not the attacking player's
        Debug.Log($"Magic Damage = {damage}");
    }

		// 서버에서만 실행되는 함수
    [Server]
    private void HealMe()
    {
        health += 10;
        Healed(10);
    }

		// 소유자에게 메시지가 전송
    [ClientRpc(target = RpcTarget.Owner)]
    public void Healed(int amount)
    {
        // No NetworkPlayer parameter, so it goes to owner
        Debug.Log($"Health increased by {amount}");
    }
}
```
