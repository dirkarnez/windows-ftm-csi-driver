windows-ftm-csi-driver
======================
<kbd>[**vscode-web-action**](https://github.com/dirkarnez/vscode-web-action/actions/workflows/vscode-web.yml)</kbd><br>

```
msbuild HelloWorldDriver.sln /t:Rebuild /p:Configuration=Release /p:Platform=x64 /p:TargetPlatformVersion=10.0.26100.0
```

### Tutorials
- [Windows-driver-samples/network/wlan at main · microsoft/Windows-driver-samples](https://github.com/microsoft/Windows-driver-samples/tree/main/network/wlan)
- [low-latency-audio/src/uac2-asio/USBAsio.sln at main · Litttlefish/low-latency-audio](https://github.com/Litttlefish/low-latency-audio/blob/main/src/uac2-asio/USBAsio.sln)

### Tools
- https://download.sysinternals.com/files/DebugView.zip

```c
OID_WDI_TASK_REQUEST_FTM

#include <ndis.h>
#include <dot11wdi.h> // 必須引用 WDK 中的 WDI 標準標頭檔

OID_WDI_GET_ADAPTER_CAPABILITIES
檢查回傳結構體中的 WDI_WLAN_CAPABILITIES。驗證其中的 FtmResponderSupported 或 FtmInitiatorSupported 布林值是否為 TRUE。

// 廠商無關的 FTM 發送邏輯
NDIS_STATUS GenericWdiFtmRequest(NDIS_HANDLE FilterModuleContext, UCHAR* TargetBssid)
{
    NDIS_STATUS status;
    PNDIS_OID_REQUEST pOidRequest = NULL;
    
    // 1. 建立一個標準的 NDIS OID 請求
    status = NdisAllocateCloneOidRequest(FilterModuleContext, NULL, 'FTMg', &pOidRequest);
    if (status != NDIS_STATUS_SUCCESS) return status;

    // 2. 填入微軟標準的 FTM 工作識別碼 (跨廠商通用)
    pOidRequest->Header.Type = NDIS_OBJECT_TYPE_OID_REQUEST;
    pOidRequest->Header.Revision = NDIS_OID_REQUEST_REVISION_1;
    pOidRequest->Header.Size = sizeof(NDIS_OID_REQUEST);
    pOidRequest->RequestType = NdisRequestMethod;
    pOidRequest->DATA.METHOD_INFORMATION.Oid = OID_WDI_TASK_REQUEST_FTM;

    // 3. 建構跨廠商通用的 WDI TLV 數據包
    // 根據微軟 WDI 規範，此處必須包含：
    // [WDI_TLV_FTM_TARGET_BSS_LIST] -> 包含 TargetBssid, Channel, Band
    // [WDI_TLV_FTM_REQUEST_PARAM]   -> 包含要求的 Burst Number 與 Timeout
    
    /* 
       實際開發時需使用 WDK 內建的 WdiGenerate... 函數：
       WDI_TASK_REQUEST_FTM_PARAMETERS params;
       // 填寫 params...
       GenerateWdiTaskRequestFtmTlv(&params, &TlvBuffer);
    */
    
    // pOidRequest->DATA.METHOD_INFORMATION.InformationBuffer = TlvBuffer;
    // pOidRequest->DATA.METHOD_INFORMATION.InputBufferLength = TlvLength;

    // 4. 下發給底層 Miniport。無論底層是 Intel 還是 MediaTek，只要支援 802.11mc，
    // 其網卡韌體都會看懂這個標準 OID，並在空中自動進行 RTT 握手與計時。
    status = NdisFOidRequest(FilterModuleContext, pOidRequest);
    
    return status;
}

VOID FilterStatus(
    NDIS_HANDLE FilterModuleContext,
    PNDIS_STATUS_INDICATION StatusIndication
)
{
    // 檢查是否為微軟標準的 FTM 完成事件 (不分廠商)
    if (StatusIndication->StatusCode == NDIS_STATUS_WDI_INDICATION_REQUEST_FTM_COMPLETE) 
    {
        // 1. 直接套用微軟官方 WDI 結構體解析
        PWDI_INDICATION_REQUEST_FTM_COMPLETE_PARAMETERS pFtmCompleteParams = 
            (PWDI_INDICATION_REQUEST_FTM_COMPLETE_PARAMETERS)StatusIndication->StatusBuffer;

        if (pFtmCompleteParams && StatusIndication->StatusBufferSize >= sizeof(WDI_INDICATION_REQUEST_FTM_COMPLETE_PARAMETERS)) 
        {
            // 2. 讀取標準 WDI FTM 結果列表
            UINT32 numEntries = pFtmCompleteParams->FtmResultList.NumberOfEntries;
            
            for (UINT32 i = 0; i < numEntries; i++) 
            {
                // 解析出跨廠商標準化的 Sample 數據
                WDI_FTM_SAMPLE_DATA sample = pFtmCompleteParams->FtmResultList.Samples[i];

                if (sample.Status == WDI_STATUS_SUCCESS) 
                {
                    // --- 這是微軟在 WDI 架構中強制定義的跨廠商標準欄位 ---
                    ULONGLONG t1_ps = sample.T1; // 統一單位：皮秒 (picoseconds)
                    ULONGLONG t2_ps = sample.T2; 
                    ULONGLONG t3_ps = sample.T3; 
                    ULONGLONG t4_ps = sample.T4; 
                    LONG rssi_dbm   = sample.Rssi; // 統一單位：dBm
                    UINT32 dist_cm  = sample.CalculatedDistance; // 統一單位：公分

                    // 這裡拿到的所有 Stuff 已經完全被 Windows WDI 抽象化
                    // 你不需要管底層硬體暫存器是 32位元 還是 64位元計時，WDI 已經統一幫你換算好了
                    PushStuffToUserModeBuffer(sample.TargetBssid, t1_ps, t2_ps, t3_ps, t4_ps, rssi_dbm, dist_cm);
                }
            }
        }
    }

    // 必須放行，交還給作業系統
    NdisFIndicateStatus(FilterModuleContext, StatusIndication);
}
```
