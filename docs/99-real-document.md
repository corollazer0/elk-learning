# real document field 
필드의 실제 결과 혹은 필드명으로 데이터의 내용을 유추할 수 있는것을 제외하고,
우리 회사에서 정의한 필드들을 따로 설명하겠습니다.

- data : 실제 request body, response body의 내용을 담은 필드
- app_uuid : 프론트엔드에서 거래 요청할 때 axios에서 채번하는 uuid
- bt_uuid : 우리는 Presentation tier, business tier로 나누어져 있으며 bt_uuid는 bt 종류의 msa로 거래할 때 채번되는 uuid
- gw_uuid : 대외 연동 시 api g/w를 통해 거래하는데 그 때 채번되는 uuid
- proc_tm : 처리 시간 ms
- svc_c : "FA"는 금융을 의미하며, BU, CD, FA, MB , PU, PC, PY, PP, WP, PA, AS 총 11개의 MSA가 있으며 각 혜택, 카드, 금융, 회원, 이용내역, 공통, 간편결제(페이), App PT, Web PT, 관리자, ACS(결제시스템)을 의미함.
- msg_c : 메세지 코드 "000000000"는 정상, 그 외 에러
- dgtl_cusno : 고객번호 key
- caller_uuid : pt는 항상 bt를 호출하고 bt는 항상 계정계코어 연동을 위해 MCI를 호출하는데 그 때 서로 호출하는 관계를 시각적으로 그려낼 때 사용하는 호출 uuid
- sts_c : 정상거래 여부 "OK" 정상, 그 외 "ERROR"
- mci_uuid : MCI 거래시 채번되는 uuid
- biz_c : 각 MSA 내부의 세부 업무코드
- chan_c : "MA", "MW", "PW", "DA"가 있으며 모바일앱, 모바일웹, pcWEB, 디지털 ARS 의미
- os_ver : 폰의 os 버전
- os_kdc : os 구분 I 아이폰, A 안드로이드, W 웹
- session_id : was에서 생성한 session id
- cd_cusno : 고객번호 key, dgtl_cusno와 함께 유니크한 key
- fir_err : 최초 에러 여부, PT->BT에서 연동 시 BT에서 에러가 발생하면 BT거래에 fir_err "Y" 표기
- tier_c : "PT", "BT", "MCI" 3종류로 나뉘어짐
- svc_id : 거래 ID springboot의 API Mapping에 연결되는 api거나, CORE에 연동하는 ID
ex) ppmc0408pC/changedGiftcdPW, NCDP_NBNC_MIMEINS3810
- scrn_uuid : 라우터를 통해 신규 화면으로 진입하면 vue.js 화면에서 채번되는 번호로 화면 내 모든 거래를 묶을 수 있는 거래 단위 키
- log_div : "*_IN", "*_OUT" 두 종류로 인 아웃을 의미
- ctnr_nm : paas 컨테이너 id
- scrn_id : 화면 id
- guid : 사용자 액션에 의헤 프론트엔드에서 서버로 요청이 최초 PT layer로 전송될때 backend전체 거래를 묶을 수 있는 key

이하 예시
```json
{
  "_index": "test-keep-ing-mcs-v1-2026.04.29",
  "_id": "1p0u2hM2JXrHeuky2",
  "_version": 1,
  "_score": 0,
  "_source": {
    "metadata": {
      "lsh_hst_nm": "nbnclolowsu2",
      "logdate": {
        "ap_out_time": "2026-04-29T16:07:37.990+09:00",
        "logstash_in_time": "2026-04-29T16:07:37.994+09:00",
        "logstash_out_time": "2026-04-29T16:07:48.116+09:00"
      }
    },
    "takes": {
      "ap_logstash_takes": 4,
      "logstash_process_takes": 0
    },
    "usr_ip": "112.170.140.95",
    "log_tm": "2026-04-29T16:07:37.990+09:00",
    "device_model": "iPhone 12 Pro Max",
    "scrn_nm": "금융메인",
    "data": {
      "MIMEINS3810_OUT": {
        "io_col_coic_agr_yn": "1",
        "ap_push_agr_yn": "1",
        "cd_rg_mbdy_no": "*****",
        "cd_rg_mbdy_dsc": "2",
        "chsv_us_agr_yn": "1",
        "cd_wb_dsc": "1"
      }
    },
    "app_uuid": "c15d3b63b6fabb8aaa43afa6d73147e",
    "bt_uuid": "2604291607370NCDPOBTFA0136367981",
    "ap_ver": "4.5.1",
    "gu_uuid": null,
    "proc_tm": 50,
    "svc_c": "FA",
    "msg_c": "00000000",
    "dgtl_cusno": "D17514365",
    "@version": "1",
    "caller_uuid": "2604291607370NCDPOBTFA0136367981",
    "sts_c": "OK",
    "mci_uuid": "2604291607370NCDPOBTFA013992081",
    "biz_c": "ST",
    "chan_c": "MA",
    "doc_sz": 963,
    "os_ver": "18.7.8",
    "os_kdc": "I",
    "session_id": "dc3cef25-dd58-499a-8597-0bdf4ae8b889",
    "cd_cusno": "338570094",
    "f1r_err": "N",
    "f1r_c": "MCI",
    "tier_c": "MCI",
    "svc_id": "NCDP_NBNC_MIMEINS3810",
    "scrn_uuid": "FN1000000F-e83c6eb19cb844076970c744bcae4706d",
    "ex": {},
    "@timestamp": "2026-04-29T16:07:37.990+09:00",
    "log_div": "MCI_RECV",
    "ctnr_nm": "7D047896-4179-01",
    "scrn_id": "FN1000000F",
    "guid": "2604291607370NCDPOPTP9833096501"
  }
}
```