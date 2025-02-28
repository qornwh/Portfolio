# 클라이언트

## 인벤토리의 smart_ptr사용

- 자신의 Gold, EquipItem, EtcItem을 관리하는 InventoryItem 클래스입니다.
- 해당하는 아이템들이 업데이트 될때 장착, 획득 메서드등이 호출됩니다.
```cpp
class ARPG_CLIENT_API InventoryItem : public TSharedFromThis<InventoryItem>
{
public:
	 ...
	bool ItemEquipped(int32 invenPos, int32 equipPos);
	void AddGold(int32 gold);
	void UseGold(int32 gold);
	void SetGold(int32 gold);
	void RemoveEquipItem(int32 Invenpos);
	void UseEtcItem(int32 Invenpos, int32 Count);
	void EquippedItem(int Invenpos, int Equippos);

	TArray<EquipItem>& GetInventoryEquipItemList() { return InventoryEquipItemList; }
	TArray<EtcItem>& GetInventoryEtcItemList() { return InventoryEtcItemList; }
	TArray<EquipItem>& GetEquippedItemList() { return EquippedItemList; }
	int32 GetGold() { return Gold; }
private:
	TArray<EquipItem> InventoryEquipItemList;
	TArray<EtcItem> InventoryEtcItemList;
	TArray<EquipItem> EquippedItemList;
	int32 Gold = 0;
	...
};
```

- 아이템 장착요청 성공여부를 호출받는 함수입니다.
- 먼저 ```나의 인벤토리```(mode->GetMyInventory())의 아이템이 장착 업데이트 됩니다.
- 다음으로 ```인벤토리UI의 아이템들이 장착 업데이트``` 됩니다.

```cpp
void ABJS_SocketActor::UpdateItemsHandler(BYTE* Buffer, PacketHeader* Header, int32 Offset)
{
	protocol::CUpdateItems pkt;

	if (PacketHandlerUtils::ParsePacketHandler(pkt, Buffer, Header->GetSize() - Offset, Offset))
	{
		auto mode = Cast<ABJS_InGameMode>(GetWorld()->GetAuthGameMode());
		if (mode)
		{
			int invenpos = pkt.invenpos();
			int equippos = pkt.equippos();
			
			mode->GetMyInventory()->EquippedItem(invenpos, equippos); // 아이템 장착 스왑
			mode->UpdateInventoryEquipUI(invenpos, 1);
			mode->UpdateEquippedItemUI(equippos, 1);
		}
	}
}
```

## 언리얼에서 효율적인 데이터 관리를 위한 DataTable사용

캐릭터 기본 능력치, 아이템 정보, 스킬 정보등의 정적인 데이터를 코드에 의존성을 줘서 수정하고 컴파일 하는 번거로움 대신 편하게 수정할수 있도록 DataTable사용을 사용했습니다.
- 데이터를 cvs로 관리하고, 이를 불러와 DataTable로드 해서 사용합니다.

```cpp
struct FItemEquipStruct : public FTableRowBase
{
	...
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Code", MakeStructureDefaultValue="0"))
	int32 Code;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Type", MakeStructureDefaultValue="0"))
	int32 Type;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Gold", MakeStructureDefaultValue="0"))
	int32 Gold;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Attack", MakeStructureDefaultValue="0"))
	int32 Attack;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Speed", MakeStructureDefaultValue="0"))
	int32 Speed;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Name", MakeStructureDefaultValue=""))
	FName Name;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Path", MakeStructureDefaultValue=""))
	FName Path;
	
	UPROPERTY(BlueprintReadWrite, EditAnywhere, meta=(DisplayName="Description", MakeStructureDefaultValue=""))
	FName Description;
};
```

데이터 로드 
- cvs에서 로드 된 데이터를 다음과 같이 1번 읽어들입니다.

```cpp
void UBJS_GameInstance::LoadDataTable()
{
	...
	static ConstructorHelpers::FObjectFinder<UDataTable> DT_ItemEquip(TEXT("/Script/Engine.DataTable'/Game/MyGame/Data/DT_ItemEquip.DT_ItemEquip'"));
  ...
	DT_ItemEquip.Object->GetAllRows<FItemEquipStruct>(TEXT("GetAllRows"), ItemEquipStructs);

	for (auto item : ItemEquipStructs)
	{
		int32 Code = item->Code;	
		ItemEquipStructMap.Add(Code, item);
	...
}
```

## 네트워크

- 서버 - 클라이언트간 어떤 패킷인지 바로 확인하기 위해 protobuf에서 지원되는 ```열거형```을 이용했습니다.

```protobuf
syntax = "proto3";

package protocol;
enum MessageCode {
    LOGIN = 0;
    S_LOAD = 1; // 유저들 정보 전달
    S_INSERTPLAYER = 2; // 추가되는 유저 정보
    S_MOVE = 3; // 이동 
    S_CHAT = 4; // 채팅
    S_PLAYERDATA = 5; // 내 정보(추가) 
    S_CLOSEPLAYER = 6; // 유저 종료
    S_UNITSTATES = 7; // 여러 유닛(유저, 몬스터) 상태
    C_ATTACK = 8; // 유저 공격
...
}
```

- read, write 버퍼같은 경우는 서버와 다르게 여러군데 보낼 이유는 없기에 ```1개씩 할당```했습니다.
- 가장 중요했던 서버로부터 ```패킷을 읽어들일 시점```은 socketActor가 맵에 배치된 시점을 선택했습니다.
- 이유는 게임맵에 사용자가 있어야지 구매, 캐릭터 선택, 캐릭터들의 움직임들을 바로 적용할수 있는 시점이었기 때문입니다.
- socket객체는 GameInstance에 1개만 생성해두고 SocketActor에서 받아와서 사용하는 방식으로 구현했습니다.

```cpp
class ARPG_CLIENT_API ABJS_SocketActor : public AActor
{
  ...
	FSocket* MySocket = nullptr;
	BJS_BufferPtr ReadBuffer;
	BJS_BufferPtr WriteBuffer;
};
...

void ABJS_SocketActor::BeginPlay()
{
	Super::BeginPlay();

	auto myInstance = Cast<UBJS_GameInstance>(GetGameInstance());
	if (myInstance)
	{
		if (!myInstance->GetIsConnect())
			myInstance->SocketConnect();
		
		MySocket = myInstance->GetSocket();
    ...
	}
  ...
}
```