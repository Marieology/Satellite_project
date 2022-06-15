# Satellite_project

# 2022.first semester

Goal > SAR data에서 water-body detection & classification 



# 1] py -> sentinel-1A/B data Download and PRE-proccessing

1) DATA DOWNLOAD
download link
- alask : https://search.asf.alaska.edu/#/
- sentinel-1 : https://scihub.copernicus.eu/dhus/#/home

download option
- search type : Geogrphic Search
- Dataset : Sentinel-1
- Beam Mode : IW
- Polarization : vv + vh (둘 다 필요)
- file type : grd

2) PRE-proccessing
step
![image](https://user-images.githubusercontent.com/95207627/173530976-21101211-df43-482f-88bc-f3a2faf4ba22.png)
![image](https://user-images.githubusercontent.com/95207627/173531109-f095cfc7-9890-4864-8dad-7e09842e5120.png)

[ 01_downloaded_data and tools ]
 - image data : S1A_IW_GRDH_1SDV_20150501T092218_20150501T092243_005726_007596_AF9E.zip
               
 - snap : https://step.esa.int/main/download/snap-download/
          sanp 설치 후 환경변수 path 추가 
          
[ 02_SNAP_setting (xml 생성 및 수정) ]
 1. graph builder(icon) click
 2. 방사보정 : Radar --> Radiometric --> Calibration
 3. 지형보정 : Radar --> Ellipsoid Correction --> Ellipsoid-Correction-RD
 4. CONNECT GRAPH로 각 box들 연결
 5. 각 탭 별 클릭하여 옵션 확인 및 선택 
 6. incidience angle 생성 및 save
      
      --> myGraph.xml 파일 생성
![image](https://user-images.githubusercontent.com/95207627/173534239-5f93755f-4233-45dc-91cc-c89a82dd503e.png)

myGraph.xml 에서 입력파일명과 출력파일명을 받는 변수를 각각 ${file} 및 ${target}으로 설정할 것

다시 graph bullder돌아와서 수정된 myGraph.xml RUN
결과 파일 2개 생성 됨
 - Cal_EC.data
 - Cal_EC.dim 


<option .xml 파일 생성 단계>

~ S1_proc1_CA_TC_out_VH_VV_INCIDENCEANGLE.xml
   
    <file>${file}</file>                                  #READ의 파일 입력 부분 위와 같이 변경
    <selectedPolarisations>VH,VV</selectedPolarisations>  #VH, VV 맞는지 확인
    <sourceBands>Sigma0_VH,Sigma0_VV</sourceBand>         #sigma0_vh, vv 확인
    <file>${target}</file>                                #WRITE의 파일 입력 부분 위와 같이 변경
    <formatName>BEAM-DIMAP</formatName>                   #.DIM형식 확인 (메모리 확보)

~ S1_proc2_VH_2deciBel.xml
  
    <file>${file}</file>                                  #READ의 파일 입력 부분 위와 같이 변경
    <operator>LinearToFromdB</operaor>                    #Raster -> DATA conversion -> LinearToFromdB
    <sourceBands>Sigma0_VH</sourceBands>                  #Sigma0_VH 확인
    <file>${target}</file>                                #WRITE의 파일 입력 부분 위와 같이 변경
    
~ S1_proc2_VV_2deciBel.xml
    
    <file>${file}</file>                                  #READ의 파일 입력 부분 위와 같이 변경
    <operator>LinearToFromdB</operaor>                    #Raster -> DATA conversion -> LinearToFromdB
    <sourceBands>Sigma0_VV</sourceBands>                  #Sigma0_VV 확인
    <file>${target}</file>                                #WRITE의 파일 입력 부분 위와 같이 변경
    
~ S1_proc2_makeband_INCIDENCEANGLE.xml

    <file>${file}</file>                                  #READ의 파일 입력 부분 위와 같이 변경
    <operator>BandMaths</operaor>                         #Raster -> BandMaths
    <targetBand>                                          #name incident_angle_deg 확인
      <name>incident_angle_deg</name>
      <type>float32</type>                                  
    <expression>incidenceAngleFromEllipsoid</expression>  #expression 확인
    <file>${target}</file>                                #WRITE의 파일 입력 부분 위와 같이 변경
    
~ S1_proc3_dim2geotiff.xml

    <file>${file}</file>                                  #READ의 파일 입력 부분 위와 같이 변경
    <file>${target}</file>                                #WRITE의 파일 입력 부분 위와 같이 변경
    <formatName>GDAL-GTiff-WRITER</formatName>            #formatname 확인
   

5개의 XML 데이터 graph builder로 생성
(만들어진 거 반복 사용 가능)


[ 03_bat ]
 
 SNAP과 연동하여 방사보정 및 지형보정 영상 생성 방법 -> SNAP_CommandLine_Tutorial.PDF 참고
 GPT <SNAP 옵션 파일 경로> -Pfile=<입력파일 경로> -Ptarget=<출력파일 경로>
 
 *S1_proc1_CA_TC_out_VH_VV_INCIDENCEANGLE.xml 파일은 SNAP의 방사보정(Calibration) 및 지형보정(Ellipsoid-Correction-RD) 옵션을 저장한 것으로 
  원본 S1 파일을 xml 파일과 연계하여 방사보정 및 지형보정이 완료된 파일을 생성함
  
 *이 때 원본 → GeoTIFF 포맷으로 바로 변환 시 작업이 중단되는 등의 문제가 종종 발생하므로 BEAM-DIMAP(확장자 .dim) 파일 생성 후 GeoTIFF로 변환하여 생성하는 방향으로 구현함
 
 *GeoTIFF 파일을 생성하는 커맨드 라인 실행시 proj.db 파일을 찾을 수 없다는 에러가 발생할 경우 proj.db 파일이 포함되어 있는 경로를 환경변수로 잡아줄 것 

 <batch 파일 실행 단계>
 
  1. 방사보정, 지형보정, incidence angle 저장
  2. VH 편파 Linear scale to dB scale
  3. VV 편파 Linear scale to dB scale
  4. Incidence angle 값을 실수형 degree로 추출
  5. .dim 포맷을 .tif 포맷으로 변환
  6. .tif 파일 제외한 폴더와 파일 삭제
 
  *gpt 구동시 지형보정 과정에서 에러 발생 Unknown element 'outputComplex'
   --> 해당 xml파일에서 <outputComplex>false</outputComplex>삭제하고 저장
  
[ 04_SNAP_output ]
 tif 파일 생성


# 2] r -> 원하는 구역 crop


# 3] py -> binary classification & gaussian filtering classification
