SimpliFi uses Async architecture to let the clients use its APIs in a modular set-up. Every API call made to SimpliFi
would result in response code in the **2xx** series, if the request was successfully received. Once the request is
received it gets handled internally to get it processed.

Once the request is processed, the response of the request is provided to the clients at their configurable endpoints
through **webhooks**. Webhook body would provide appropriate values in case the request was completed as well as error
codes in case the request resulted in failure.

### Understanding API Resources

- **CardProduct**: This is the highest level of the resource hierarchy with-in SimpliFi platform. `CardProduct` are
  implemented at the partnering processors depending upon client requirements and approvals from regulatory bodies
  and schemes. `CardProduct` implementation once completed would return a `CardProgramTemplateUUID` which can be
  used to create card programs.

- **Funding Source**: Every client (or company i.e. client of SimpliFi client) would get a dedicated funding source
  in each approved currencies for `CardProduct`. One funding source can be linked with multiple Cards Products or
  Card Programs. Funding source is the Account number that is created to specifically run the card programs created
  under a Company.

- **CardProgram**: These are configured templates and define the unique use case per client. These can also be
  configured and provided dedicated names to identify unique company in case the SimpliFi client is not the end
  client issuing the cards to its customers. Any client can have multiple Card Program using the same template
  (product type) or different template (product type).

- **User**: Users are cardholders to whom the cards would be issued. Users can be created using APIs. At the time of
  user creation; users are assigned under a card program. During card issuance for the users the cardprogram can be
  overwritten by providing a different card program as an input. If card program as an input during card issuance is
  not provided then the default card program used during user creation can be used.

- **Card**: These are payment cards which can be issued in virtual or physical form and can be used to conduct
  transactions. One user can have multiple cards.

- **Wallet/Currency**: SimpliFi supports multi currency environment. Currently the SimpliFi platform
  supports 50+ currencies. If the card requires more than one currency on the card, a dedicated wallet would be
  created supporting each currency. Supported currencies are listed below.

- **PCI DSS Compliant SDK**: Both iOS and Android SDKs which can be wrapped under client App to enable
  cardholders access their card details on the app.

- **Transactions**: Transactions on SimpliFi platform could be accessed using two mechanisms:
    - **API**: The API can be called to see all the transactions at Card, User or Card Program level. However
      this API would not be enough to update the client system with real time transactions.
    - **Webhook**: To support clients update their system with real time transactions (which the card holders would
      be conducting outside of their system), SimpliFi would send the webhooks with all the transactional details.

#### Supported Currencies
<Columns>
  <Column>
    AED ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/aed.svg)
  </Column>
  <Column>
    BHD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/bhd.svg)
  </Column>
  <Column>
    SAR ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/sar.svg)
  </Column>
  <Column>
    JOD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/jod.svg)
  </Column>
</Columns>
<Columns>
  <Column>
    KWD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/kwd.svg)
  </Column>
  <Column>
    OMR ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/omr.svg)
  </Column>
  <Column>
    QAR ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/qar.svg)
  </Column>
</Columns>
<Columns>
  <Column>
    AUD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/aud.svg)
  </Column>
  <Column>
    CAD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/cad.svg)
  </Column>
  <Column>
    EUR ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/eur.svg)
  </Column>
</Columns>
<Columns>
  <Column>
    GBP ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/gbp.svg)
  </Column>
  <Column>
    USD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/usd.svg)
  </Column>
  <Column>
    HKD ![](https://raw.githubusercontent.com/Lissy93/currency-flags/master/assets/flags_svg/hkd.svg)
  </Column>
</Columns>


### Getting API Keys

To request API keys for the SimpliFi platform please place a [request here](mailto:sales@simplifipay.com). Once
we receive your request we’d be able to send you:
 - Client ID
 - User ID
 - Temporary Password
 - Dummy funding source in USD currency
 - CardProgramTemplateUUID

You will be able to use the first three credentials to start testing APIs (or access SimpliFi Portal). You can use
the dummy funding source to link it with the card programs you would have created. We would also be setting up
a `CardProgramTemplateUUID` for you to start creating a card program.
