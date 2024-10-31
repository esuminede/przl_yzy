# przl_yzy 
#bilgraf lab h3 pürüzlü yüzey üretimi

// Deney uygulaması ve tasarımı

# Adım 1:
cpp dosyası ConstantBuffer structına int key; değişkenini ekledik.
#Adım 2:
cpp 254. satırda 	case 'E': bloğundan sonra 0, 1, 2, 3 durumlarını ekledik.

    case '0':
        cb.key = 0;
        break;
    
    case '1':
        cb.key = 1;
        break;

    case '2':
        cb.key = 2;
        break;

    case '3':
        cb.key = 3;
        break;
}

#Adım 3:
.fx dosyasnda 12. satırdaki ConstantBuffer içine int key satırını ekledik. 

#Adım4: BU KISMI TAM ANLAYAMADIM. DOĞRULUĞUNU TEYİT ETMELİ.
.fx dosyasında satır 100'deki fParallaxLimit değerini 0 yaparak parallax işlemini kaldırabiliyoruz. 
	if(key == 0){
		vFinalNormal = float4(0.0, 0.0, 1.0, 0.0);
		fParallaxLimit = 0;
	}

	else if(key == 1){
		vFinalNormal = float4(0.0, 0.0, 1.0, 0.0);
		fParallaxLimit  *= fHeightMapScale;
	}

	else if(key ==2){
		vFinalNormal = vFinalNormal * 0.2f - 1.0f;
		fParallaxLimit = 0;












