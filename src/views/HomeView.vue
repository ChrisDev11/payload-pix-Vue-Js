<template>
  <div>
    <h1>Gerador de Payload Pix</h1>

    <!-- Formulário para inserir informações -->
    <form @submit.prevent="generatePayload">
      <label for="pixKey">Chave Pix:</label>
      <input type="text" id="pixKey" v-model="formData.pixKey" required>

      <label for="amount">Valor:</label>
      <input type="number" id="amount" v-model="formData.amount" required>

      <label for="uniquePayment">Pagamento único:</label>
      <input type="checkbox" id="uniquePayment" v-model="formData.uniquePayment">

      <label for="merchantName">Nome do titular:</label>
      <input type="text" id="merchantName" v-model="formData.merchantName" required>

      <label for="merchantCity">Cidade do titular:</label>
      <input type="text" id="merchantCity" v-model="formData.merchantCity" required>

      <label for="txid">ID da transação:</label>
      <input type="text" id="txid" v-model="formData.txid" required>

      <button type="submit">Gerar Payload</button>
    </form>

    <!-- Exibição do Payload Pix gerado -->
    <div v-if="payloadGenerated">
      <h2>Payload Pix Gerado:</h2>
      <pre>{{ payloadGenerated }}</pre>
    </div>
  </div>
</template>

<script>
// Define a classe Payload
class Payload {
  // IDs do Payload do Pix
  static ID_PAYLOAD_FORMAT_INDICATOR = '00';
  static ID_POINT_OF_INITIATION_METHOD = '01';
  static ID_MERCHANT_ACCOUNT_INFORMATION = '26';
  static ID_MERCHANT_ACCOUNT_INFORMATION_GUI = '00';
  static ID_MERCHANT_ACCOUNT_INFORMATION_KEY = '01';
  static ID_MERCHANT_ACCOUNT_INFORMATION_DESCRIPTION = '02';
  static ID_MERCHANT_ACCOUNT_INFORMATION_URL = '25';
  static ID_MERCHANT_CATEGORY_CODE = '52';
  static ID_TRANSACTION_CURRENCY = '53';
  static ID_TRANSACTION_AMOUNT = '54';
  static ID_COUNTRY_CODE = '58';
  static ID_MERCHANT_NAME = '59';
  static ID_MERCHANT_CITY = '60';
  static ID_ADDITIONAL_DATA_FIELD_TEMPLATE = '62';
  static ID_ADDITIONAL_DATA_FIELD_TEMPLATE_TXID = '05';
  static ID_CRC16 = '63';

  // Construtor da classe
  constructor() {
    this.pixKey = '';
    this.description = '';
    this.merchantName = '';
    this.merchantCity = '';
    this.txid = '';
    this.amount = '';
    this.uniquePayment = false;
    this.url = '';
  }

  // Define o valor de pixKey
  setPixKey(pixKey) {
    this.pixKey = pixKey;
    return this;
  }

  // Define o valor de uniquePayment
  setUniquePayment(uniquePayment) {
    this.uniquePayment = uniquePayment;
    return this;
  }

  // Define o valor de url
  setUrl(url) {
    this.url = url;
    return this;
  }

  // Define o valor de description
  setDescription(description) {
    this.description = description;
    return this;
  }

  // Define o valor de merchantName
  setMerchantName(merchantName) {
    this.merchantName = merchantName;
    return this;
  }

  // Define o valor de merchantCity
  setMerchantCity(merchantCity) {
    this.merchantCity = merchantCity;
    return this;
  }

  // Define o valor de txid
  setTxid(txid) {
    this.txid = txid;
    return this;
  }

  // Define o valor de amount
  setAmount(amount) {
    this.amount = amount.toFixed(2);
    return this;
  }

  // Retorna o valor completo de um objeto do payload
  getValue(id, value) {
    const size = value.length.toString().padStart(2, '0');
    return id + size + value;
  }

  // Retorna os valores completos da informação da conta
  getMerchantAccountInformation() {
    // DOMÍNIO DO BANCO
    const gui = this.getValue(
      Payload.ID_MERCHANT_ACCOUNT_INFORMATION_GUI,
      'br.gov.bcb.pix'
    );

    // CHAVE PIX
    const key =
      this.pixKey.length > 0
        ? this.getValue(Payload.ID_MERCHANT_ACCOUNT_INFORMATION_KEY, this.pixKey)
        : '';

    // DESCRIÇÃO DO PAGAMENTO
    const description =
      this.description.length > 0
        ? this.getValue(
            Payload.ID_MERCHANT_ACCOUNT_INFORMATION_DESCRIPTION,
            this.description
          )
        : '';

    // URL DO QR CODE DINÂMICO
    const url =
      this.url.length > 0
        ? this.getValue(
            Payload.ID_MERCHANT_ACCOUNT_INFORMATION_URL,
            this.url.replace(/^https?:\/\//, '')
          )
        : '';

    // VALOR COMPLETO DA CONTA
    return this.getValue(
      Payload.ID_MERCHANT_ACCOUNT_INFORMATION,
      gui + key + description + url
    );
  }

  // Retorna os valores completos do campo adicional do pix (TXID)
  getAdditionalDataFieldTemplate() {
    // TXID
    const txid = this.getValue(
      Payload.ID_ADDITIONAL_DATA_FIELD_TEMPLATE_TXID,
      this.txid
    );

    // RETORNA O VALOR COMPLETO
    return this.getValue(
      Payload.ID_ADDITIONAL_DATA_FIELD_TEMPLATE,
      txid
    );
  }

  // Retorna o valor do ID_POINT_OF_INITIATION_METHOD
  getUniquePayment() {
    return this.uniquePayment
      ? this.getValue(Payload.ID_POINT_OF_INITIATION_METHOD, '12')
      : '';
  }

  // Gera o código completo do payload Pix
  getPayload() {
    // CRIA O PAYLOAD
    const payload =
      this.getValue(Payload.ID_PAYLOAD_FORMAT_INDICATOR, '01') +
      this.getUniquePayment() +
      this.getMerchantAccountInformation() +
      this.getValue(Payload.ID_MERCHANT_CATEGORY_CODE, '0000') +
      this.getValue(Payload.ID_TRANSACTION_CURRENCY, '986') +
      this.getValue(Payload.ID_TRANSACTION_AMOUNT, this.amount) +
      this.getValue(Payload.ID_COUNTRY_CODE, 'BR') +
      this.getValue(Payload.ID_MERCHANT_NAME, this.merchantName) +
      this.getValue(Payload.ID_MERCHANT_CITY, this.merchantCity) +
      this.getAdditionalDataFieldTemplate();

    // RETORNA O PAYLOAD + CRC16
    return payload + this.getCRC16(payload);
  }

  // Calcula o valor da hash de validação do código pix
  getCRC16(payload) {
    // ADICIONA DADOS GERAIS NO PAYLOAD
    payload += Payload.ID_CRC16 + '04';

    // DADOS DEFINIDOS PELO BACEN
    const polinomio = 0x1021;
    let resultado = 0xFFFF;

    // CHECKSUM
    if (payload.length > 0) {
      for (let offset = 0; offset < payload.length; offset++) {
        resultado ^= (payload.charCodeAt(offset) << 8);
        for (let bitwise = 0; bitwise < 8; bitwise++) {
          if (((resultado <<= 1) & 0x10000)) {
            resultado ^= polinomio;
          }
          resultado &= 0xFFFF;
        }
      }
    }

    // RETORNA CÓDIGO CRC16 DE 4 CARACTERES
    return (
      Payload.ID_CRC16 + '04' + resultado.toString(16).toUpperCase().padStart(4, '0')
    );
  }
}

export default {
  data() {
    return {
      formData: {
        pixKey: '',
        amount: 0,
        uniquePayment: false,
        merchantName: '',
        merchantCity: '',
        txid: '',
      },
      payloadGenerated: null,
    };
  },
  methods: {
    generatePayload() {
      // Crie uma instância da classe Payload e configure os dados a partir do formulário
      const payload = new Payload();
      payload
        .setPixKey(this.formData.pixKey)
        .setAmount(this.formData.amount)
        .setUniquePayment(this.formData.uniquePayment)
        .setMerchantName(this.formData.merchantName)
        .setMerchantCity(this.formData.merchantCity)
        .setTxid(this.formData.txid);

      // Gere o payload Pix
      const generatedPayload = payload.getPayload();

      // Atualize o estado para exibir o payload gerado
      this.payloadGenerated = generatedPayload;
    },
  },
};
</script>

<style scoped>
/* Adicione estilos CSS conforme necessário */
</style>
