
# Secured Information Management

## Keystore

To save sensitive data on Android, the best way is using Keystore.

Sample Heler Class :

```java

import android.content.Context;
import android.os.Build;
import android.security.KeyPairGeneratorSpec;
import android.security.keystore.KeyGenParameterSpec;
import android.security.keystore.KeyProperties;
import android.util.Base64;
import android.util.Log;

import java.math.BigInteger;
import java.nio.charset.StandardCharsets;
import java.security.KeyPairGenerator;
import java.security.KeyStore;
import java.security.spec.AlgorithmParameterSpec;
import java.util.Calendar;
import java.util.GregorianCalendar;

import javax.crypto.Cipher;
import javax.security.auth.x500.X500Principal;

public class SecuredSPwithKeyStore {
    private final String TAG = SecuredSPwithKeyStore.class.getSimpleName();
    private static final String KEY_STORE = "AndroidKeyStore";
    private static final String ALGORITHM = "RSA";
    public static final String ALIAS = "MapleSharedPreferenceKey";
    private static final String TRANSFORMATION = "RSA/ECB/PKCS1Padding";
    private static final String PROVIDER_LESS_M = "AndroidOpenSSL";
    private static final String PROVIDER_MORE_M = "AndroidKeyStoreBCWorkaround";

    private Context mContext;
    private KeyStore mKeyStore;

    /**
     * constructor
     */
    public SecuredSPwithKeyStore(Context context) {
        mContext = context;
    }

    /**
     * check the key in Keystore
     */
    public void initKeyStore() {

        try {
            mKeyStore = KeyStore.getInstance(KEY_STORE);
            mKeyStore.load(null);

            if (!mKeyStore.containsAlias(ALIAS)) {
                createKeys();
            }

        } catch (Exception e) {
            Log.e(TAG, "initKeyStore Exception: " + e.getMessage());
        }
    }

    /**
     * Creates a public and private key and stores it using the Android Key Store
     */
    public void createKeys() {
        try {
            // Create a start and end time, for the validity range of the key pair that's about
            // to be generated.
            Calendar start = new GregorianCalendar();
            Calendar end = new GregorianCalendar();
            end.add(Calendar.YEAR, 1);

            // Initialize a KeyPair generator using the the intended algorithm (in this example, RSA
            // and the KeyStore.  This example uses the AndroidKeyStore.
            KeyPairGenerator kpGenerator = KeyPairGenerator.getInstance(ALGORITHM, KEY_STORE);

            // The KeyPairGeneratorSpec object is how parameters for your key pair are passed
            // to the KeyPairGenerator.
            AlgorithmParameterSpec spec;

            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
                // Below Android M, use the KeyPairGeneratorSpec.Builder.

                spec = new KeyPairGeneratorSpec.Builder(mContext)
                        // You'll use the alias later to retrieve the key.  It's a key for the key!
                        .setAlias(ALIAS)
                        // The subject used for the self-signed certificate of the generated pair
                        .setSubject(new X500Principal("CN=" + ALIAS))
                        // The serial number used for the self-signed certificate of the
                        // generated pair.
                        .setSerialNumber(BigInteger.ONE)
                        // Date range of validity for the generated pair.
                        .setStartDate(start.getTime())
                        .setEndDate(end.getTime())
                        .build();

            } else {
                // On Android M or above, use the KeyGenparameterSpec.Builder and specify permitted
                // properties  and restrictions of the key.
                spec = new KeyGenParameterSpec.Builder(ALIAS,
                        KeyProperties.PURPOSE_DECRYPT | KeyProperties.PURPOSE_ENCRYPT)
                        .setCertificateSubject(new X500Principal("CN=" + ALIAS))
                        .setCertificateSerialNumber(BigInteger.ONE)
                        .setCertificateNotBefore(start.getTime())
                        .setCertificateNotAfter(end.getTime())
                        .setBlockModes(KeyProperties.BLOCK_MODE_ECB)
                        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_RSA_PKCS1)
                        .build();

            }

            kpGenerator.initialize(spec);

        } catch (Exception e) {
            Log.e(TAG, "createKeys Exception : " + e.getMessage());
        }
    }

    /**
     * Encryption
     */
    public String doEncrypt(String secret) {
        initKeyStore();

        byte[] vals = null;

        try {
            Cipher inputCipher = getCipher();

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                inputCipher.init(Cipher.ENCRYPT_MODE,
                        mKeyStore.getCertificate(ALIAS).getPublicKey());
            } else {
                KeyStore.PrivateKeyEntry privateKeyEntry =
                        (KeyStore.PrivateKeyEntry) mKeyStore.getEntry(ALIAS, null);
                inputCipher.init(Cipher.ENCRYPT_MODE,
                        privateKeyEntry.getCertificate().getPublicKey());
            }

            vals = inputCipher.doFinal(secret.getBytes());

        } catch (Exception e) {
            Log.e(TAG, "rsaEncrypt Exception : " + e.getMessage());
        }

        return Base64.encodeToString(vals, Base64.DEFAULT);
    }

    /**
     * Decryption
     */
    public String doDecrypt(String encrypted) {
        initKeyStore();

        String decryptStr = "";
        try {

            Cipher output = getCipher();
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                output.init(Cipher.DECRYPT_MODE, mKeyStore.getKey(ALIAS, null));

            } else {
                KeyStore.PrivateKeyEntry privateKeyEntry =
                        (KeyStore.PrivateKeyEntry) mKeyStore.getEntry(ALIAS, null);
                output.init(Cipher.DECRYPT_MODE, privateKeyEntry.getPrivateKey());
            }

            byte[] decode = Base64.decode(encrypted, Base64.DEFAULT);
            byte[] decodedBytes = output.doFinal(decode);

            decryptStr = new String(decodedBytes, StandardCharsets.UTF_8);
        } catch (Exception e) {
            Log.e(TAG, "rsaDecrypt Exception : " + e.getMessage());
        }

        return decryptStr;
    }

    /**
     * return Cipher
     */
    private Cipher getCipher() {
        Cipher cipher = null;
        try {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) { // below android m
                cipher = Cipher.getInstance(TRANSFORMATION, PROVIDER_LESS_M);
                // error in android 6: InvalidKeyException: Need RSA private or public key
            } else { // android m and above
                cipher = Cipher.getInstance(TRANSFORMATION, PROVIDER_MORE_M);
                // error in android 5: NoSuchProviderException: Provider not available:
                // AndroidKeyStoreBCWorkaround
            }
        } catch (Exception e) {
            Log.e(TAG, "getCipher Exception : " + e.getMessage());
        }
        return cipher;
    }

```
