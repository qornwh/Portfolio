# Web-API 서버

## ASP 서버
 - 모든 api는는 필요한 파라미터나 응답 데이터들은 스펙이 변경이나 크기가 너무 큰 문제가 있었습니다.
   - 이런 문제를 고려해 get(parameter)이 아닌 ```post(body)```로 구성되어 있습니다.
 - 2가지의 다른 테이블의 쿼리가 동시에 성공적으로 작동해야 되는 경우 트랜잭션 사용 했습니다.[(코드 링크)](https://github.com/qornwh/MMO_GameServer/blob/9fc45c864d072dcb50d9c2410183c6b1ae976958/MMO_ApiServer_ASP/MMO_ApiServer_ASP/Controllers/Service/AccountService.cs#L189)

## C++ 서버와 연동
 - 게임서버에서 API서버에 요청하기 위해서 ```cpr```라이브러리를 사용했습니다.
 - 응답, 요청시 데이터교환시 ```가독성```을 위한 ```DTO 클래스``` 구현했습니다.
 - API 서버

```cpp
  public class AccountRequest
  {
      public int AccountCode { get; set; }
  }

  public class AccountResponse
  {
      public int Ret { get; set; }
      public AccountDTO Account { get; set; }
  }
  public class AccountDTO
  {
      public int AccountCode { get; set; }
      public int Cash { get; set; }
      public int WeaponOne { get; set; }
      public int WeaponTwo { get; set; }
      public int WeaponThr { get; set; }
      public int CurPlayerType { get; set; }
      public int CurWeaponType { get; set; }
  }
```

 - 게임 서버
   - post의 body에 데이터를 넣기 위한 json작성을 위해 ```nlohmann::json``` 라이브러리를 사용했습니다.

```cpp
  std::string AccountRequest::requestStr(const int accountCode)
  {
    nlohmann::json json;
    json["accountCode"] = accountCode;
    return json.dump();
  }

  bool AccountResponse::response(cpr::Response& res)
  {
  	nlohmann::json bodyData = GameUtils::JsonParser::GetStrParser(res.text);
  	_ret = bodyData["ret"].get<int32>();
  	if (_ret == 1)
  	{
  		nlohmann::json accountJson = GameUtils::JsonParser::Parser("account", bodyData);
  		_account.Dto(accountJson);
  		return true;
  	}
  	return false;
  }

  void AccountDTO::Dto(nlohmann::json data)
  {
  	accountCode = data["accountCode"].get<int32>();
  	cash = data["cash"].get<int32>();
  	weaponOne = data["weaponOne"].get<int32>();
  	weaponTwo = data["weaponTwo"].get<int32>();
  	weaponThr = data["weaponThr"].get<int32>();
  	curPlayerType = data["curPlayerType"].get<int32>();
  	curWeaponType = data["curWeaponType"].get<int32>();
  }
```
