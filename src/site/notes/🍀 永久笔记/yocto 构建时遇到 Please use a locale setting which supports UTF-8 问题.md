---
{"dg-publish":true,"permalink":"/🍀 永久笔记/yocto 构建时遇到 Please use a locale setting which supports UTF-8 问题/"}
---


```shell
u@728c4b2a9f61:/home/data/vmware/os/nxp/L4.19.35_imx8qm/build-xwayland$ bitbake zht-image  
Please use a locale setting which supports UTF-8 (such as LANG=en_US.UTF-8).  
Python can't change the filesystem locale after loading so we need a UTF-8 when Python starts or things won't work.
```

这个是本地语言设置的问题：python 需要本地语言环境为 en_US.UTF-8

解决方法如下：

#### 先安装语言包

`sudo apt-get install locales`

#### 设置语言

`sudo dpkg-reconfigure locales`

使用上述命令后会弹出，选择 `149 en_US.UTF-8, 3. en_US.UTF-8` 

```shell
u@728c4b2a9f61:/home/data/vmware/os/nxp/L4.19.35_imx8qm/build-xwayland$ sudo dpkg-reconfigure locales  
perl: warning: Setting locale failed.  
perl: warning: Please check that your locale settings:  
       LANGUAGE = (unset),  
       LC_ALL = (unset),  
       LANG = "en_US.UTF-8"  
   are supported and installed on your system.  
perl: warning: Falling back to the standard locale ("C").  
debconf: unable to initialize frontend: Dialog  
debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 76.)  
debconf: falling back to frontend: Readline  
Configuring locales  
-------------------  
  
Locales are a framework to switch between multiple languages and allow users to use their language, country, characters, collation order, etc.  
  
Please choose which locales to generate. UTF-8 locales should be chosen by default, particularly for new installations. Other character sets may be useful for backwards  
compatibility with older systems and software.  
  
 1. All locales                  98. da_DK.UTF-8 UTF-8               195. es_US.UTF-8 UTF-8              292. ks_IN UTF-8             389. shs_CA UTF-8  
 2. aa_DJ ISO-8859-1             99. de_AT ISO-8859-1                196. es_UY ISO-8859-1               293. ks_IN@devanagari UTF-8  390. si_LK UTF-8  
 3. aa_DJ.UTF-8 UTF-8            100. de_AT.UTF-8 UTF-8              197. es_UY.UTF-8 UTF-8              294. ku_TR ISO-8859-9        391. sid_ET UTF-8  
 4. aa_ER UTF-8                  101. de_AT@euro ISO-8859-15         198. es_VE ISO-8859-1               295. ku_TR.UTF-8 UTF-8       392. sk_SK ISO-8859-2  
 5. aa_ER@saaho UTF-8            102. de_BE ISO-8859-1               199. es_VE.UTF-8 UTF-8              296. kw_GB ISO-8859-1        393. sk_SK.UTF-8 UTF-8  
 6. aa_ET UTF-8                  103. de_BE.UTF-8 UTF-8              200. et_EE ISO-8859-1               297. kw_GB.UTF-8 UTF-8       394. sl_SI ISO-8859-2  
 7. af_ZA ISO-8859-1             104. de_BE@euro ISO-8859-15         201. et_EE.ISO-8859-15 ISO-8859-15  298. ky_KG UTF-8             395. sl_SI.UTF-8 UTF-8  
 8. af_ZA.UTF-8 UTF-8            105. de_CH ISO-8859-1               202. et_EE.UTF-8 UTF-8              299. lb_LU UTF-8             396. so_DJ ISO-8859-1  
 9. ak_GH UTF-8                  106. de_CH.UTF-8 UTF-8              203. eu_ES ISO-8859-1               300. lg_UG ISO-8859-10       397. so_DJ.UTF-8 UTF-8  
 10. am_ET UTF-8                 107. de_DE ISO-8859-1               204. eu_ES.UTF-8 UTF-8              301. lg_UG.UTF-8 UTF-8       398. so_ET UTF-8  
 11. an_ES ISO-8859-15           108. de_DE.UTF-8 UTF-8              205. eu_ES@euro ISO-8859-15         302. li_BE UTF-8             399. so_KE ISO-8859-1  
 12. an_ES.UTF-8 UTF-8           109. de_DE@euro ISO-8859-15         206. eu_FR ISO-8859-1               303. li_NL UTF-8             400. so_KE.UTF-8 UTF-8  
 13. anp_IN UTF-8                110. de_LI.UTF-8 UTF-8              207. eu_FR.UTF-8 UTF-8              304. lij_IT UTF-8            401. so_SO ISO-8859-1  
 14. ar_AE ISO-8859-6            111. de_LU ISO-8859-1               208. eu_FR@euro ISO-8859-15         305. ln_CD UTF-8             402. so_SO.UTF-8 UTF-8  
 15. ar_AE.UTF-8 UTF-8           112. de_LU.UTF-8 UTF-8              209. fa_IR UTF-8                    306. lo_LA UTF-8             403. sq_AL ISO-8859-1  
 16. ar_BH ISO-8859-6            113. de_LU@euro ISO-8859-15         210. ff_SN UTF-8                    307. lt_LT ISO-8859-13       404. sq_AL.UTF-8 UTF-8  
 17. ar_BH.UTF-8 UTF-8           114. doi_IN UTF-8                   211. fi_FI ISO-8859-1               308. lt_LT.UTF-8 UTF-8       405. sq_MK UTF-8  
 18. ar_DZ ISO-8859-6            115. dv_MV UTF-8                    212. fi_FI.UTF-8 UTF-8              309. lv_LV ISO-8859-13       406. sr_ME UTF-8  
 19. ar_DZ.UTF-8 UTF-8           116. dz_BT UTF-8                    213. fi_FI@euro ISO-8859-15         310. lv_LV.UTF-8 UTF-8       407. sr_RS UTF-8  
 20. ar_EG ISO-8859-6            117. el_CY ISO-8859-7               214. fil_PH UTF-8                   311. lzh_TW UTF-8            408. sr_RS@latin UTF-8  
 21. ar_EG.UTF-8 UTF-8           118. el_CY.UTF-8 UTF-8              215. fo_FO ISO-8859-1               312. mag_IN UTF-8            409. ss_ZA UTF-8  
 22. ar_IN UTF-8                 119. el_GR ISO-8859-7               216. fo_FO.UTF-8 UTF-8              313. mai_IN UTF-8            410. st_ZA ISO-8859-1  
 23. ar_IQ ISO-8859-6            120. el_GR.UTF-8 UTF-8              217. fr_BE ISO-8859-1               314. mg_MG ISO-8859-15       411. st_ZA.UTF-8 UTF-8  
 24. ar_IQ.UTF-8 UTF-8           121. en_AG UTF-8                    218. fr_BE.UTF-8 UTF-8              315. mg_MG.UTF-8 UTF-8       412. sv_FI ISO-8859-1  
 25. ar_JO ISO-8859-6            122. en_AU ISO-8859-1               219. fr_BE@euro ISO-8859-15         316. mhr_RU UTF-8            413. sv_FI.UTF-8 UTF-8  
 26. ar_JO.UTF-8 UTF-8           123. en_AU.UTF-8 UTF-8              220. fr_CA ISO-8859-1               317. mi_NZ ISO-8859-13       414. sv_FI@euro ISO-8859-15  
 27. ar_KW ISO-8859-6            124. en_BW ISO-8859-1               221. fr_CA.UTF-8 UTF-8              318. mi_NZ.UTF-8 UTF-8       415. sv_SE ISO-8859-1  
 28. ar_KW.UTF-8 UTF-8           125. en_BW.UTF-8 UTF-8              222. fr_CH ISO-8859-1               319. mk_MK ISO-8859-5        416. sv_SE.ISO-8859-15 ISO-8859-15  
 29. ar_LB ISO-8859-6            126. en_CA ISO-8859-1               223. fr_CH.UTF-8 UTF-8              320. mk_MK.UTF-8 UTF-8       417. sv_SE.UTF-8 UTF-8  
 30. ar_LB.UTF-8 UTF-8           127. en_CA.UTF-8 UTF-8              224. fr_FR ISO-8859-1               321. ml_IN UTF-8             418. sw_KE UTF-8  
 31. ar_LY ISO-8859-6            128. en_DK ISO-8859-1               225. fr_FR.UTF-8 UTF-8              322. mn_MN UTF-8             419. sw_TZ UTF-8  
 32. ar_LY.UTF-8 UTF-8           129. en_DK.ISO-8859-15 ISO-8859-15  226. fr_FR@euro ISO-8859-15         323. mni_IN UTF-8            420. szl_PL UTF-8  
 33. ar_MA ISO-8859-6            130. en_DK.UTF-8 UTF-8              227. fr_LU ISO-8859-1               324. mr_IN UTF-8             421. ta_IN UTF-8  
 34. ar_MA.UTF-8 UTF-8           131. en_GB ISO-8859-1               228. fr_LU.UTF-8 UTF-8              325. ms_MY ISO-8859-1        422. ta_LK UTF-8  
 35. ar_OM ISO-8859-6            132. en_GB.ISO-8859-15 ISO-8859-15  229. fr_LU@euro ISO-8859-15         326. ms_MY.UTF-8 UTF-8       423. tcy_IN.UTF-8 UTF-8  
 36. ar_OM.UTF-8 UTF-8           133. en_GB.UTF-8 UTF-8              230. fur_IT UTF-8                   327. mt_MT ISO-8859-3        424. te_IN UTF-8  
 37. ar_QA ISO-8859-6            134. en_HK ISO-8859-1               231. fy_DE UTF-8                    328. mt_MT.UTF-8 UTF-8       425. tg_TJ KOI8-T  
 38. ar_QA.UTF-8 UTF-8           135. en_HK.UTF-8 UTF-8              232. fy_NL UTF-8                    329. my_MM UTF-8             426. tg_TJ.UTF-8 UTF-8  
 39. ar_SA ISO-8859-6            136. en_IE ISO-8859-1               233. ga_IE ISO-8859-1               330. nan_TW UTF-8            427. th_TH TIS-620  
[More]    
  
 40. ar_SA.UTF-8 UTF-8           137. en_IE.UTF-8 UTF-8              234. ga_IE.UTF-8 UTF-8              331. nan_TW@latin UTF-8      428. th_TH.UTF-8 UTF-8  
 41. ar_SD ISO-8859-6            138. en_IE@euro ISO-8859-15         235. ga_IE@euro ISO-8859-15         332. nb_NO ISO-8859-1        429. the_NP UTF-8  
 42. ar_SD.UTF-8 UTF-8           139. en_IN UTF-8                    236. gd_GB ISO-8859-15              333. nb_NO.UTF-8 UTF-8       430. ti_ER UTF-8  
 43. ar_SS UTF-8                 140. en_NG UTF-8                    237. gd_GB.UTF-8 UTF-8              334. nds_DE UTF-8            431. ti_ET UTF-8  
 44. ar_SY ISO-8859-6            141. en_NZ ISO-8859-1               238. gez_ER UTF-8                   335. nds_NL UTF-8            432. tig_ER UTF-8  
 45. ar_SY.UTF-8 UTF-8           142. en_NZ.UTF-8 UTF-8              239. gez_ER@abegede UTF-8           336. ne_NP UTF-8             433. tk_TM UTF-8  
 46. ar_TN ISO-8859-6            143. en_PH ISO-8859-1               240. gez_ET UTF-8                   337. nhn_MX UTF-8            434. tl_PH ISO-8859-1  
 47. ar_TN.UTF-8 UTF-8           144. en_PH.UTF-8 UTF-8              241. gez_ET@abegede UTF-8           338. niu_NU UTF-8            435. tl_PH.UTF-8 UTF-8  
 48. ar_YE ISO-8859-6            145. en_SG ISO-8859-1               242. gl_ES ISO-8859-1               339. niu_NZ UTF-8            436. tn_ZA UTF-8  
 49. ar_YE.UTF-8 UTF-8           146. en_SG.UTF-8 UTF-8              243. gl_ES.UTF-8 UTF-8              340. nl_AW UTF-8             437. tr_CY ISO-8859-9  
 50. as_IN UTF-8                 147. en_US ISO-8859-1               244. gl_ES@euro ISO-8859-15         341. nl_BE ISO-8859-1        438. tr_CY.UTF-8 UTF-8  
 51. ast_ES ISO-8859-15          148. en_US.ISO-8859-15 ISO-8859-15  245. gu_IN UTF-8                    342. nl_BE.UTF-8 UTF-8       439. tr_TR ISO-8859-9  
 52. ast_ES.UTF-8 UTF-8          149. en_US.UTF-8 UTF-8              246. gv_GB ISO-8859-1               343. nl_BE@euro ISO-8859-15  440. tr_TR.UTF-8 UTF-8  
 53. ayc_PE UTF-8                150. en_ZA ISO-8859-1               247. gv_GB.UTF-8 UTF-8              344. nl_NL ISO-8859-1        441. ts_ZA UTF-8  
 54. az_AZ UTF-8                 151. en_ZA.UTF-8 UTF-8              248. ha_NG UTF-8                    345. nl_NL.UTF-8 UTF-8       442. tt_RU UTF-8  
 55. be_BY CP1251                152. en_ZM UTF-8                    249. hak_TW UTF-8                   346. nl_NL@euro ISO-8859-15  443. tt_RU@iqtelif UTF-8  
 56. be_BY.UTF-8 UTF-8    bian liang       153. en_ZW ISO-8859-1               250. he_IL ISO-8859-8               347. nn_NO ISO-8859-1        444. ug_CN UTF-8  
 57. be_BY@latin UTF-8           154. en_ZW.UTF-8 UTF-8              251. he_IL.UTF-8 UTF-8              348. nn_NO.UTF-8 UTF-8       445. ug_CN@latin UTF-8  
 58. bem_ZM UTF-8                155. eo ISO-8859-3                  252. hi_IN UTF-8                    349. nr_ZA UTF-8             446. uk_UA KOI8-U  
 59. ber_DZ UTF-8                156. eo.UTF-8 UTF-8                 253. hne_IN UTF-8                   350. nso_ZA UTF-8            447. uk_UA.UTF-8 UTF-8  
 60. ber_MA UTF-8                157. eo_US.UTF-8 UTF-8              254. hr_HR ISO-8859-2               351. oc_FR ISO-8859-1        448. unm_US UTF-8  
 61. bg_BG CP1251                158. es_AR ISO-8859-1               255. hr_HR.UTF-8 UTF-8              352. oc_FR.UTF-8 UTF-8       449. ur_IN UTF-8  
 62. bg_BG.UTF-8 UTF-8           159. es_AR.UTF-8 UTF-8              256. hsb_DE ISO-8859-2              353. om_ET UTF-8             450. ur_PK UTF-8  
 63. bhb_IN.UTF-8 UTF-8          160. es_BO ISO-8859-1               257. hsb_DE.UTF-8 UTF-8             354. om_KE ISO-8859-1        451. uz_UZ ISO-8859-1  
 64. bho_IN UTF-8                161. es_BO.UTF-8 UTF-8              258. ht_HT UTF-8                    355. om_KE.UTF-8 UTF-8       452. uz_UZ.UTF-8 UTF-8  
 65. bn_BD UTF-8                 162. es_CL ISO-8859-1               259. hu_HU ISO-8859-2               356. or_IN UTF-8             453. uz_UZ@cyrillic UTF-8  
 66. bn_IN UTF-8                 163. es_CL.UTF-8 UTF-8              260. hu_HU.UTF-8 UTF-8              357. os_RU UTF-8             454. ve_ZA UTF-8  
 67. bo_CN UTF-8                 164. es_CO ISO-8859-1               261. hy_AM UTF-8                    358. pa_IN UTF-8             455. vi_VN UTF-8  
 68. bo_IN UTF-8                 165. es_CO.UTF-8 UTF-8              262. hy_AM.ARMSCII-8 ARMSCII-8      359. pa_PK UTF-8             456. wa_BE ISO-8859-1  
 69. br_FR ISO-8859-1            166. es_CR ISO-8859-1               263. ia_FR UTF-8                    360. pap_AN UTF-8            457. wa_BE.UTF-8 UTF-8  
 70. br_FR.UTF-8 UTF-8           167. es_CR.UTF-8 UTF-8              264. id_ID ISO-8859-1               361. pap_AW UTF-8            458. wa_BE@euro ISO-8859-15  
 71. br_FR@euro ISO-8859-15      168. es_CU UTF-8                    265. id_ID.UTF-8 UTF-8              362. pap_CW UTF-8            459. wae_CH UTF-8  
 72. brx_IN UTF-8                169. es_DO ISO-8859-1               266. ig_NG UTF-8                    363. pl_PL ISO-8859-2        460. wal_ET UTF-8  
 73. bs_BA ISO-8859-2            170. es_DO.UTF-8 UTF-8              267. ik_CA UTF-8                    364. pl_PL.UTF-8 UTF-8       461. wo_SN UTF-8  
 74. bs_BA.UTF-8 UTF-8           171. es_EC ISO-8859-1               268. is_IS ISO-8859-1               365. ps_AF UTF-8             462. xh_ZA ISO-8859-1  
 75. byn_ER UTF-8                172. es_EC.UTF-8 UTF-8              269. is_IS.UTF-8 UTF-8              366. pt_BR ISO-8859-1        463. xh_ZA.UTF-8 UTF-8  
 76. ca_AD ISO-8859-15           173. es_ES ISO-8859-1               270. it_CH ISO-8859-1               367. pt_BR.UTF-8 UTF-8       464. yi_US CP1255  
 77. ca_AD.UTF-8 UTF-8           174. es_ES.UTF-8 UTF-8              271. it_CH.UTF-8 UTF-8              368. pt_PT ISO-8859-1        465. yi_US.UTF-8 UTF-8  
 78. ca_ES ISO-8859-1            175. es_ES@euro ISO-8859-15         272. it_IT ISO-8859-1               369. pt_PT.UTF-8 UTF-8       466. yo_NG UTF-8  
 79. ca_ES.UTF-8 UTF-8           176. es_GT ISO-8859-1               273. it_IT.UTF-8 UTF-8              370. pt_PT@euro ISO-8859-15  467. yue_HK UTF-8  
 80. ca_ES.UTF-8@valencia UTF-8  177. es_GT.UTF-8 UTF-8              274. it_IT@euro ISO-8859-15         371. quz_PE UTF-8            468. zh_CN GB2312  
 81. ca_ES@euro ISO-8859-15      178. es_HN ISO-8859-1               275. iu_CA UTF-8                    372. raj_IN UTF-8            469. zh_CN.GB18030 GB18030  
 82. ca_ES@valencia ISO-8859-15  179. es_HN.UTF-8 UTF-8              276. iw_IL ISO-8859-8               373. ro_RO ISO-8859-2        470. zh_CN.GBK GBK  
 83. ca_FR ISO-8859-15           180. es_MX ISO-8859-1               277. iw_IL.UTF-8 UTF-8              374. ro_RO.UTF-8 UTF-8       471. zh_CN.UTF-8 UTF-8  
 84. ca_FR.UTF-8 UTF-8           181. es_MX.UTF-8 UTF-8              278. ja_JP.EUC-JP EUC-JP            375. ru_RU ISO-8859-5        472. zh_HK BIG5-HKSCS  
 85. ca_IT ISO-8859-15           182. es_NI ISO-8859-1               279. ja_JP.UTF-8 UTF-8              376. ru_RU.CP1251 CP1251     473. zh_HK.UTF-8 UTF-8  
 86. ca_IT.UTF-8 UTF-8           183. es_NI.UTF-8 UTF-8              280. ka_GE GEORGIAN-PS              377. ru_RU.KOI8-R KOI8-R     474. zh_SG GB2312  
[More]    
  
 87. ce_RU UTF-8                 184. es_PA ISO-8859-1               281. ka_GE.UTF-8 UTF-8              378. ru_RU.UTF-8 UTF-8       475. zh_SG.GBK GBK  
 88. ckb_IQ UTF-8                185. es_PA.UTF-8 UTF-8              282. kk_KZ PT154                    379. ru_UA KOI8-U            476. zh_SG.UTF-8 UTF-8  
 89. cmn_TW UTF-8                186. es_PE ISO-8859-1               283. kk_KZ RK1048                   380. ru_UA.UTF-8 UTF-8       477. zh_TW BIG5  
 90. crh_UA UTF-8                187. es_PE.UTF-8 UTF-8              284. kk_KZ.UTF-8 UTF-8              381. rw_RW UTF-8             478. zh_TW.EUC-TW EUC-TW  
 91. cs_CZ ISO-8859-2            188. es_PR ISO-8859-1               285. kl_GL ISO-8859-1               382. sa_IN UTF-8             479. zh_TW.UTF-8 UTF-8  
 92. cs_CZ.UTF-8 UTF-8           189. es_PR.UTF-8 UTF-8              286. kl_GL.UTF-8 UTF-8              383. sat_IN UTF-8            480. zu_ZA ISO-8859-1  
 93. csb_PL UTF-8                190. es_PY ISO-8859-1               287. km_KH UTF-8                    384. sc_IT UTF-8             481. zu_ZA.UTF-8 UTF-8  
 94. cv_RU UTF-8                 191. es_PY.UTF-8 UTF-8              288. kn_IN UTF-8                    385. sd_IN UTF-8  
 95. cy_GB ISO-8859-14           192. es_SV ISO-8859-1               289. ko_KR.EUC-KR EUC-KR            386. sd_IN@devanagari UTF-8  
 96. cy_GB.UTF-8 UTF-8           193. es_SV.UTF-8 UTF-8              290. ko_KR.UTF-8 UTF-8              387. sd_PK UTF-8  
 97. da_DK ISO-8859-1            194. es_US ISO-8859-1               291. kok_IN UTF-8                   388. se_NO UTF-8  
  
(Enter the items you want to select, separated by spaces.)  
  
Locales to be generated: 149  
  
Many packages in Debian use locales to display text in the correct language for the user. You can choose a default locale for the system from the generated locales.  
  
This will select the default language for the entire system. If this system is a multi-user system where not all users are able to speak the default language, they will  
experience difficulties.  
  
 1. None  2. C.UTF-8  3. en_US.UTF-8  
Default locale for the system environment: 3  
  
Generating locales (this might take a while)...  
 en_US.UTF-8... done  
Generation complete.
```

#### 更新环境变量

`sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8`