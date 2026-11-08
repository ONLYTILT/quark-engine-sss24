# Совместимость с Screen Space Shaders Update 24

Изменения ветки `sss24-compat` относительно оригинального Quark Engine **v1.5.0** (коммит `01ec062`, влит merge-коммитом `09fb8d3`; изначально ветка делалась от v1.0.2, `3a8c8ad`).

Цель — чтобы мод **Screen Space Shaders Update 24** (Ascii1457, RC3) работал на Quark без своего exe. Апскейлеры (DLSS/FSR) и Reflex не реализованы: движок рендерит в выходном разрешении.

Исходников exe, которые идут с SSS 24, нет в открытом доступе. Они собраны из локальной копии monolith 2026.10.11. Всё ниже восстановлено по шейдерам мода, строкам и дизассемблированному коду этих exe. Со старыми версиями SSS движок по-прежнему совместим: новые проходы включаются, только если в gamedata есть шейдеры SSS 24.

## Шейдерные скрипты (`.s`)

- **`afterupscaling(bool)`** — новая функция компилятора. В SSS 24 ей помечены PDA, коллиматоры и подсветка оружия: «рисовать после апскейла». Без апскейлера она ведёт себя как прежний `distort(bool)`. Без неё эти шейдеры не компилировались.
  `xrRenderDX10/dx10ResourceManager_Scripting.cpp`
- **`dx10Options():getLevel()`** больше не падает, если уровень ещё не загружен (возвращает пустую строку). SSS 24 вызывает её в `ssfx_water.s`, а движок компилирует этот шейдер при создании устройства.
  `xrRenderDX10/dx10ResourceManager_Scripting.cpp`

## Цели рендера

- **Алиасы новых имён.** Без апскейлера они совпадают со старыми целями:

  | Имя в SSS 24 | Цель Quark |
  |---|---|
  | `$user$scene_aa` | `$user$generic0` |
  | `$user$upscaled_heat` | `$user$heat` |
  | `$user$upscaled_pos` | `$user$generic2` |

  `xrRenderDX10/dx10ResourceManager_Resources.cpp` (`_CreateTexture`)
- **`$user$scene_final`** — отдельная цель в формате generic-целей.
  - Перед отрисовкой 3D-прицелов в неё копируется сцена (`phase_3DSSReticle`), это картинка за линзой.
  - Перед постпроцессом копируется готовый кадр (`postprocess.s` читает её вместо `albedo`).
  - Без первого копирования линза прицела была чёрной, когда постпроцесс выключен.

  `r4_rendertarget.cpp`, `r4_rendertarget_phase_combine.cpp`, `rendertarget_phase_nightvision.cpp`
- **Волны воды и рябь** (размеры и форматы как в exe мода):

  | Цель | Размер | Формат |
  |---|---|---|
  | `$user$ssfx_water_waves` | 256×256 | `A8R8G8B8`, **с mip-уровнями** |
  | `$user$ssfx_water_waves_tmp` | 256×256 | `A8R8G8B8` |
  | `$user$ssfx_rain_ripples` | 512×512 | `A16B16G16R16F` |

  `r2_types.h`, `r4_rendertarget.h`, `r4_rendertarget.cpp`
- **Цели рендера с мипами.** У `CRT::create`, `resptrcode_crt::create` и `CResourceManager::_CreateRT` появился необязательный параметр `useMips`: полная цепочка mip-уровней и `D3D11_RESOURCE_MISC_GENERATE_MIPS`. Он сохраняется и при пересоздании устройства.
  `xrRender/SH_RT.h`, `xrRender/ResourceManager.h`, `xrRenderDX10/dx10SH_RT.cpp`, `xrRenderDX10/dx10ResourceManager_Resources.cpp`

## Вода

- **Проход волн** (`phase_ssfx_water_waves`), если в gamedata есть `ssfx_water_waves_build.ps`:
  1. `ssfx_water_waves.ps` рисует высоты в `ssfx_water_waves_tmp`;
  2. `ssfx_water_waves_build.ps` собирает из них нормали в `ssfx_water_waves` (`water_n0`/`water_n1` = `fx\water_normal`);
  3. `ssfx_rain_ripples.ps` генерирует рябь от дождя в `ssfx_rain_ripples`;
  4. для `ssfx_water_waves` вызывается `GenerateMips`. Без мипов мелкий узор волн вдали мерцал, и отражения дёргались даже на спокойной воде.

  Viewport берётся из размера цели. Новые элементы блендера: `E[3]` — сборка нормалей, `E[4]` — рябь.
  `xrRender/rendertarget_phase_blur.cpp`, `xrRenderPC_R4/blender_blur.cpp`
- **Рефракция.** Перед финальной отрисовкой воды текущая сцена копируется в `$user$generic_temp` (`s_rimage`), из которого шейдер воды берёт всё, что видно сквозь поверхность. Раньше там лежал более старый кадр, уже с водой.
  `r4_rendertarget_phase_combine.cpp`
- **Motion vectors воды.** При отрисовке воды буфер `ssfx_motion_vectors` подключается в слот 2. В него шейдер воды SSS 24 пишет скорость и метку материала «вода».
  `r4_rendertarget_phase_combine.cpp`
- **Flowmap уровня.** При загрузке уровня шейдер `ssfx_water` пересоздаётся, чтобы `.s` подхватил `water\<уровень>_flowmap`. Раньше он создавался один раз до загрузки уровня, и течения не было нигде.
  `xrRenderPC_R4/r4_loader.cpp`

## Константы шейдеров

- **`m_SMAP_res`** = (`smapsize`, `smapsize`, 1/`smapsize`, 1/`smapsize`). На ней построена фильтрация теней в `shadow.h` SSS 24. Без неё было деление на ноль.
- **`ssfx_flowmap_setup`** — масштаб и смещение текстуры течения, задаётся одноимённой консольной командой.

`xrRender/Blender_Recorder_StandartBinding.cpp`

## HUD и motion vectors

Все HUD-проходы теперь переключают проекцию прошлого кадра (`set_xform_project_prev`) вместе с текущей и потом восстанавливают обе. Это проходы sorted HUD (включая объекты, прикреплённые к камере), emissive HUD, distort HUD, 3DSS и HUD UI.

Раньше прошлая проекция оставалась мировой. Motion vectors PDA в SSS 24 получались радиальными, и TAA/motion blur размазывали экран PDA.

`xrRender/r__dsgraph_render.cpp`

## Консольные команды

| Команда | Что делает |
|---|---|
| `ssfx_flowmap_setup x,y,z,w` | Масштаб (xy) и смещение (zw) flowmap. Выставляет `ssfx_underground_check.script`. |
| `ssfx_cascades_resolution (a,b,c,0)` | Только хранит значение, отдельного разрешения для каждого каскада в Quark нет. |
| `ssfx_cascades_delay (a,b,c,0)` | Только хранит значение. |
| `ssfx_upscaler off\|dlss\|fsr` | Только хранит значение, апскейлеров нет. |
| `ssfx_upscaler_resolution <режим>` | Только хранит значение. |
| `ssfx_upscaler_sharp <0..2>` | Только хранит значение. |

Команды-заглушки нужны, чтобы скрипты MCM SSS 24 и `user.ltx` работали без ошибок «неизвестная команда».

`xrRender/xrRender_console.cpp`

## Не реализовано

- DLSS / FSR (Streamline, `sl.*.dll`) и всё, что с ними связано: внутреннее разрешение, `upscaled_*_pv`, `ssfx_upscale_pos*`, `ssfx_dlss_preset`, `ssfx_upscaler_automipmap`.
- Команда мода `ssfx_reflex`. В самом Quark с v1.5.0 есть свой режим низкой задержки (`r__low_latency off|on|reflex|reflex_boost`), он работает независимо от SSS 24.
- Отдельные разрешение и задержка обновления для каждого каскада теней.
- Маска `$user$3dssmask` и проходы `ssfx_3dss_mask` / `ssfx_3dss_border`.
- Инстансинг травы (`detail_buffer`, exe «MT - Grass Instancing»).
- Константы льда (`ssfx_ice*`): модуля льда в SSS 24 RC3 нет.

## Замечания по установке мода

- **Не выбирать опцию архива «EX - Grass Instancing».** Её `details_blend.s` и `deffer_detail_*_flat.vs` читают траву из `StructuredBuffer`, а это работает только с exe «MT - Grass Instancing». На Quark загрузка уровня падает в `R_constant_table::parse`. Если опция уже установлена, нужные файлы вернёт мод `GR. ScreenSpaceShaders_24 hotfix 1`. Его приоритет в MO2 должен быть выше, чем у Update 24.
- **Flowmap есть для 14 уровней, а таблица `ssfx_underground_check.script` задаёт `ssfx_flowmap_setup` только для 6:** `k00_marsh`, `l10_red_forest`, `l10_limansk`, `pripyat`, `jupiter`, `zaton`. На остальных значение остаётся от предыдущего уровня. Это особенность мода, в его собственном exe так же.
- После обновления exe стоит очистить `appdata\shaders_cache`.
