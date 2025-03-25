# Financeiro com Stripe

**Depois de uma pesquisa na documentação do Stripe, a solução mais viável é a Stripe connect Custom.**


_Uma conta conectada Custom é praticamente invisível para o titular da conta. Você (a plataforma) é responsável por todas as
interações com suas contas conectadas e pela coleta de todos os dados necessários para verificar cada conta.
Com contas conectadas Custom, você pode modificar os dados e configurações da conta conectada usando a API, incluindo o gerenciamento das contas bancárias e do cronograma de repasses. Como os titulares de contas conectadas Custom não conseguem acessar a Stripe, você é responsável por criar o fluxo de onboarding, o painel da conta conectada, a funcionalidade de geração de relatórios e os canais de comunicação._


## Connect Express vs Custom

**A opção Custom foi escolhida porque oferece total controle sobre a experiência do usuário. Com ela, a plataforma gerencia o onboarding, a coleta de dados, a verificação de identidade (KYC) e o painel de usuário. Isso é essencial para personalizar a experiência e adaptar o fluxo de pagamentos de acordo com as necessidades específicas da plataforma. Ao contrário do Express, que limita a personalização e exige que os usuários acessem um painel da Stripe, a Custom permite criar um processo 100% integrado e personalizado para o seu sistema de apostas**


### Criar um usuario com a subconta personalizada

```javascript

async function criarContaConectada(email) {
    const account = await stripe.accounts.create({
      type: "custom",
      country: "BR",
      email: email,
      capabilities: {
        transfers: { requested: true },
        card_payments: { requested: true }
      },
      business_type: "individual",
    });
  
    console.log("Conta criada:", account.id);
    return account;
  }

```

### Atualizar subconta
**Essa função atualiza os dados da subconta. Sem essa atualização, o Stripe não consegue realizar o saque**

```javascript

async function atualizarDadosKYC(accountId) {
    const account = await stripe.accounts.update(accountId, {
      individual: {
        first_name: "nome",
        last_name: "sobreNome",
        id_number: "123.456.789-00",
        dob: { day: 10, month: 5, year: 1995 },
        address: {
          line1: "Rua Exemplo, 123",
          city: "São Paulo",
          state: "SP",
          postal_code: "01000-000",
          country: "BR"
        }
      }
    });
  
    console.log("Dados KYC atualizados:", account.id);
  }

```

### Fazer depósito

```javascript

async function criarDepositoComPix(valor, usuarioStripeId) {
    const paymentIntent = await stripe.paymentIntents.create({
        amount: valor,
        currency: "brl",
        payment_method_types: ['pix'],
        transfer_data: {
            destination: usuarioStripeId, 
        },
    });

    console.log(`Pagamento de R$${valor / 100} criado para o usuário ${usuarioStripeId}!`);
    return paymentIntent;
}

```

### Consultar saldo da subcota do usuário

```javascript

async function verificarSaldoUsuario(usuarioStripeId) {
  try {
   
    const saldo = await stripe.balance.retrieve({
      stripeAccount: usuarioStripeId, // ID da conta conectada do usuário
    });

    
    const saldoDisponivel = saldo.available[0]?.amount || 0;
    
    console.log(`Saldo disponível para o usuário ${usuarioStripeId}: R$${(saldoDisponivel / 100).toFixed(2)}`);
    return saldoDisponivel / 100; // Converte centavos para reais
  } catch (error) {
    console.error("Erro ao verificar saldo do usuário:", error);
    throw error;
  }
}

```

### Descontar taxa

```javascript

async function descontar50Centavos(usuarioStripeId) {
    try {
      const charge = await stripe.charges.create({
        amount: 50, // R$0,50 em centavos
        currency: 'brl',
        application_fee_amount: 50, // Mantém o valor dentro da plataforma
        transfer_data: {
          destination: usuarioStripeId, // ID da conta do usuário na Stripe
        },
        description: "Desconto de R$0,50 para início da partida",
      });
  
      console.log(`R$0,50 descontado do saldo da conta ${usuarioStripeId}: ${charge.id}`);
      return charge;
    } catch (error) {
      console.error("Erro ao descontar R$0,50:", error);
      throw error;
    }
  }

```

### Processar aposta

```javascript

async function processarAposta(vencedorId, perdedorId, valor) {
  try {
    const valorCentavos = valor * 100; // Converter reais para centavos

    
    const pagamento = await stripe.charges.create({
      amount: valorCentavos,
      currency: 'brl',
      source: null, 
      description: `Desconto de R$${valor.toFixed(2)} do perdedor (${perdedorId}) para aposta`,
      application_fee_amount: 0, // Plataforma não cobra taxa adicional
      transfer_data: {
        destination: vencedorId, 
      },
    }, {
      stripeAccount: perdedorId, 
    });

    console.log(`✅ Aposta processada: ${perdedorId} perdeu R$${valor.toFixed(2)} para ${vencedorId}.`);
    return pagamento;
  } catch (error) {
    console.error("❌ Erro ao processar a aposta:", error);
    throw error;
  }
}

```

## Testes das funções Stripe

#### Configuração da conta conectada
- [x] Criar um usuário com a subconta personalizada e retonar id
- [x] Atualizar os dados KYC da subconta  

#### Gerenciamento financeiro  
- [ ] Fazer depósito via Pix  
- [x] Consultar saldo da subconta do usuário  

#### Apostas e transações  
- [ ] Descontar taxa de R$0,50 no final da partida(desconto sendo aplicado no prêmio)
- [ ] Processar aposta e transferir prêmio ao vencedor  

