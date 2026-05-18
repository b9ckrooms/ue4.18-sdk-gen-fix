# ue4.18-sdk-gen-fix

Fix for Unreal Engine 4.18 SDK generators to prevent crashes by reading CDO directly by offset 248!

UE 4.18 Mobile SDK Gen Fix (ARM64)

В чем проблема?

Обычно генераторы SDK для поиска дефолтных параметров классов дергают виртуальный метод GetDefaultObjectIndex через VTable по индексу 114 offset 0x390

В мобильных билдах под iOS/Android ARM64 из-за жесткой оптимизации компилятора и динамической сборки таблиц в памяти этот вызов стабильно отправляет дампер в краш Segmentation Fault или генерирует пустые файлы структур

Решение -> Без VTable

При реверсе UClass::Serialize было найдено прямое статическое смещение поля ClassDefaultObject (CDO) внутри структуры UClass. Оно составляет 248 байт -> 0xF8

Вместо капризных виртуальных вызовов можно читать данные прямо из памяти. Это работает в разы быстрее и вообще не падает

[ UClass ] ──► +0xF8 (248) ──► [ ClassDefaultObject ] ──► +0xC (12) ──► InternalIndex

Просто замените старый вызов через индекс 114 в вашем дампере на этот вариант ->

int32_t GetDefaultObjectIndex(uint64_t uClass) {
    if (!uClass) return 0;
    
    uint64_t cdo = *(uint64_t*)(uClass + 248); 
    if (!cdo) return 0;
    
    return *(int32_t*)(cdo + 12); 
}

What is the issue?

Typically, SDK generators call the GetDefaultObjectIndex virtual method via VTable at index 114 (offset 0x390) to retrieve default class parameters
In mobile builds (iOS/Android ARM64), aggressive compiler optimizations and dynamic VTable construction in memory cause this specific call to consistently trigger a Segmentation Fault or result in empty/broken structure files

The Solution VTable Bypass

By reverse engineering UClass::Serialize, a direct static offset for the ClassDefaultObject (CDO) field within the UClass structure was identified
It is located exactly at 248 bytes -> 0xF8

Instead of relying on fragile virtual calls, data can be read directly from memory. This approach is significantly faster and completely eliminates crashes

[ UClass ] ──► +0xF8 (248) ──► [ ClassDefaultObject ] ──► +0xC (12) ──► InternalIndex

Simply replace the old VTable index 114 call in your dumper with the following implementation:
C++

int32_t GetDefaultObjectIndex(uint64_t uClass) {
    if (!uClass) return 0;
    
    uint64_t cdo = *(uint64_t*)(uClass + 248); 
    if (!cdo) return 0;
    
    return *(int32_t*)(cdo + 12); 
}
