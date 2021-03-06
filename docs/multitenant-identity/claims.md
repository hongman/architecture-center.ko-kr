---
title: Work with claim-based identities in multitenant applications
description: How a use claims for issuer validation and authorization
author: MikeWasson
ms.service: guidance
ms.topic: article
ms.date: 05/23/2016
ms.author: pnp

pnp.series.title: Manage Identity in Multitenant Applications
pnp.series.prev: authenticate
pnp.series.next: signup
---
# 클레임 기반 ID를 이용한 작업

[![GitHub](../_images/github.png) 샘플 코드][sample application]

## Azure AD의 클레임
사용자가 로그인하면, Azure AD는 사용자에 대한 클레임 세트를 포함한 ID 토큰을 전송합니다. 클레임은 단순히 키/값의 쌍으로 표현된 정보입니다. 예: `email`=`bob@contoso.com`. 클레임은 발급자가 있습니다 - 이 경우, Azure AD - 이 발급자는 사용자를 인증하고 클레임을 생성하는 주체입니다.  발급자를 신뢰하기 때문에 클레임을 신뢰합니다. (반대로, 발급자를 신뢰하지 않는 경우, 클레임을 신뢰하면 안 됩니다!)

상위 수준에서:

1.	사용자가 인증되었습니다.
2.	IDP가 클레임 세트를 전송합니다.
3.	앱은 클레임을 일반화하거나 확대합니다(선택).
4.	앱은 인증을 결정하기 위해서 클레임을 사용합니다.


OpenID Connect에서 사용자가 받은 클레임 세트는 인증 요청의 [범위 매개변수](http://nat.sakimura.org/2012/01/26/scopes-and-claims-in-openid-connect/)에 의해서 제어됩니다.  그러나, Azure AD는 OpenID Connect를 통해서 제한된 클레임 세트를 발급합니다; [지원된 토큰과 클레임 유형](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-token-and-claims)을 참조하세요. I사용자에 관한 자세한 정보를 원하는 경우, Azure AD Graph API를 사용해야 합니다.

다음은 일반적으로 앱이 관심을 두는 AAD의 몇 가지 클레임입니다:

| ID 토큰의 클레임 유형  | 설명 |
| --- | --- |
| aud |토큰이 발급된 대상. 응용 프로그램의 클라이언트 ID입니다. 미들웨어가 클레임을 자동으로 확인하기 때문에, 이 클레임을 우려하지 않아도 됩니다. 예:  `"91464657-d17a-4327-91f3-2ed99386406f"` |
| groups |사용자가 구성원을 이루는 AAD 그룹 목록. 예:  `["93e8f556-8661-4955-87b6-890bc043c30f", "fc781505-18ef-4a31-a7d5-7d931d7b857e"]` |
| iss |OIDC 토큰의 [발급자](http://openid.net/specs/openid-connect-core-1_0.html#IDToken). 예: `https://sts.windows.net/b9bd2162-77ac-4fb2-8254-5c36e9c0a9c4/` |
| name |사용자의 표시 이름. 예: `"Alice A."` |
| oid |AAD에서 사용자의 개체 ID. 이 값은 변경이 불가능하고 다시 사용할 수 없는 사용자 ID입니다. 이 값은 이메일이 아닌, 사용자의 고유 ID로 사용합니다; 이메일 주소는 변경 가능합니다. 앱에서 Azure AD Graph API를 사용할 경우, 개체 ID는 프로필 정보를 쿼리하는데 사용되는 값입니다. 예: `"59f9d2dc-995a-4ddf-915e-b3bb314a7fa4"` |
| roles |사용자의 앱 역할 목록 예: `["SurveyCreator"]` |
| tid |테넌트 ID. Azure AD에서 이 값은 테넌트의 고유 ID입니다. 예: `"b9bd2162-77ac-4fb2-8254-5c36e9c0a9c4"` |
| unique_name |사람이 읽을 수 있는 사용자의 표시 이름 예: `"alice@contoso.com"` |
| upn |사용자 계정 이름 예: `"alice@contoso.com"` |

이 표는 ID 토큰으로 표시되는 클레임 유형에 대한 목록입니다. ASP.NET Core 1.0에서 사용자 계정에 대한 클레임 컬렉션을 채울 때 OpenID Connect 미들웨어는 일부 클레임 유형을 변환합니다. 

* oid > `http://schemas.microsoft.com/identity/claims/objectidentifier`

* tid > `http://schemas.microsoft.com/identity/claims/tenantid`

* unique_name > `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name`

* upn > `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn`

## 클레임 변환
인증 흐름 과정에서, IDP에서 받은 클레임을 수정하고 싶을 수 있습니다. ASP.NET Core 1.0에서, 사용자는 OpenID Connect 미들웨어에서 일어난 **AuthenticationValidated** 이벤트 안에서 클레임 변환을 수행할 수 있습니다. ([인증 이벤트](https://docs.microsoft.com/en-us/azure/architecture/multitenant-identity/authenticate#authentication-events)를 참조하세요.)

**AuthenticationValidated** 이벤트에서 추가한 클레임은 세션 인증 쿠키에 저장됩니다. 이 클레임들은 Azure AD로 되돌아가지 않습니다.

다음은 클레임 변환에 대한 예입니다:

* **클레임 일반화**, 즉 클레임을 사용자들 사이에서 일관되게 만들기. 이는 특히 유사한 정보에 대해서 다양한 클레임 유형을 사용할 수 있는 다중 IDP에서 클레임을 받는 경우 적절한 방법입니다. 예를 들면, Azure AD는 사용자 이메일을 포함한 "upn" 클레임을 전송합니다.  다른 IDP는 "email" 클레임을 전송할 수 있습니다. 다음 코드는 "upn" 클레임을 "email" 클레임으로 변환합니다:
  
  ```csharp
  var email = principal.FindFirst(ClaimTypes.Upn)?.Value;
  if (!string.IsNullOrWhiteSpace(email))
  {
   identity.AddClaim(new Claim(ClaimTypes.Email, email));
  }
  ```
* 존재하지 않는 **기본 클레임 값**을 추가합니다 - 예를 들면, 기본 역할에 사용자 한 명을 지정합니다.  경우에 따라, 이것은 인증 로직을 단순화할 수 있습니다.

* 사용자에 관하여 응용 프로그램별 정보가 있는 **사용자 지정 클레임 유형**을 추가합니다.  예를 들면, 데이터베이스에 사용자에 관한 정보를 저장할 수 있습니다. 이 정보가 있는 사용자 지정 클레임을 인증 티켓에 추가할 수 있습니다. 클레임이 쿠키에 저장되기 때문에, 데이터베이스에서 로그인 세션마다 한 번씩 클레임을 받기만 하면 됩니다. 다른 한편으로, 너무 큰 쿠기 생성을 피하고 싶을 수 있으므로, 쿠키 크기와 데이터베이스 검색 간 절충을 고려해야 합니다.

인증 흐름이 완료된 후, 클레임은 `HttpContext.User`에서 사용할 수 있습니다.  이 때, 클레임을 읽기 전용 컬렉션으로 처리해야 합니다 - 예: 인증 결정을 내리는 데 사용합니다.

## 발급자 확인
OpenID Connect에서, 발급자 클레임("iss")은 ID 토큰을 발급한 IDP를 식별합니다. OIDC 인증 흐름 중 하나는 발급자 클레임이 실제 발급자와 일치하는지 확인하는 것입니다. OIDC 미들웨어가 대신 이 작업을 처리합니다.

Azure AD에서, 발급자 값은 AD 테넌트별로 고유합니다(`https://sts.windows.net/<tenantID>`). 그러므로, 응용 프로그램은 그 발급자가 앱 로그인이 허용된 테넌트를 나타내는지 추가로 확인을 해야 합니다.

단일 테넌트 응용 프로그램의 경우, 발급자가 독자 소유의 테넌트인지 확인할 수 있습니다. OIDC 미들웨어는 사실상 기본값으로 이 작업을 자동으로 수행합니다. 다중 테넌트 앱에서 다양한 테넌트에 대응하는 다중 발급자를 허용해야 합니다. 다음은 일반적인 사용 방법입니다:

•	OIDC 미들웨어 옵션에서, **ValidateIssuer**를 false로 설정합니다. 이 경우 자동 확인이 해제됩니다.

•	테넌트가 등록되면, 테넌트와 발급자를 사용자 DB에 저장합니다.

•	사용자가 로그인할 때마다 데이터베이스에서 발급자를 검색합니다. 발급자가 발견되지 않으면 테넌트가 등록되지 않았다는 뜻입니다. 이 경우 테넌트를 등록 페이지로 리디렉션할 수 있습니다.

•	특정 테넌트를 블랙리스트에 올릴 수 있는데, 가령, 구독 비용을 지불하지 않은 고객의 테넌트가 해당될 수 있습니다.

더 자세한 내용은 [Sign-up and tenant onboarding in a multitenant application][signup]을 참조하세요.

## 인증을 위한 클레임 사용
클레임이 있음으로 해서 사용자 ID는 더 이상 단일체가 아닙니다. 예를 들면, 사용자는 이메일 주소, 전화번호, 생일, 성별 등을 가집니다. 아마 사용자 IDP는 이 모든 정보를 저장할 것입니다.  그런데 사용자를 인증할 때, 대개 이 정보의 부분집합을 클레임으로 받습니다. 이 모델에서 사용자 ID는 단순히 클레임의 묶음입니다. 사용자에 대한 인증을 결정할 때, 특정한 클레임 세트를 찾을 것입니다. 다시 말해서, "사용자 X가 동작 Y를 수행할 수 있나요"라는 질문은 결국 "사용자 X가 클레임 Z를 갖고 있나요"입니다.

다음은 클레임 확인을 위한 기본적인 패턴입니다.

* 사용자가 특정 값을 지닌 특정 클레임을 갖고 있는지 확인하려면:
  
   ```csharp
   if (User.HasClaim(ClaimTypes.Role, "Admin")) { ... }
   ```
  이 코드는 사용자가 그 값이 "Admin"인 Role 클레임을 갖고 있는지 확인합니다. 코드는 사용자가 Role 클레임을 갖고 있지 않거나 다중의 Role 클레임을 갖고 있는 경우를 정확히 처리합니다. 
   **클레임 유형** 클래스는 자주 사용하는 클레임 유형에 상수를 정의합니다. 그렇더라도, 클레임 유형에 문자열 값을 사용할 수 있습니다.
   
* 많아야 한 가지 값이라고 예상할 때, 클레임 유형에 단일 값을 지정하려면:
  
  ```csharp
  string email = User.FindFirst(ClaimTypes.Email)?.Value;
  ```
* To get all the values for a claim type:
  
  ```csharp
  IEnumerable<Claim> groups = User.FindAll("groups");
  ```

자세한 정보는 [다중 테넌트 응용 프로그램의 역할 기반 및 리소스 기반 인증][authorization]을 참조하세요.

[**다음**][signup]


<!-- Links -->

[scope parameter]: http://nat.sakimura.org/2012/01/26/scopes-and-claims-in-openid-connect/
[Supported Token and Claim Types]: /azure/active-directory/active-directory-token-and-claims/
[issuer]: http://openid.net/specs/openid-connect-core-1_0.html#IDToken
[Authentication events]: authenticate.md#authentication-events
[signup]: signup.md
[Claims-Based Authorization]: https://docs.asp.net/en/latest/security/authorization/claims.html
[sample application]: https://github.com/Azure-Samples/guidance-identity-management-for-multitenant-apps
[authorization]: authorize.md
