# test
import com.prowidesoftware.swift.model.mx.MxPain00100112;
import com.prowidesoftware.swift.model.mx.MxPacs00900107;
import com.prowidesoftware.swift.model.mx.dic.*; // Prowide dictionary for ISO 20022
import java.util.*;

public class ISOMessageConverter {

    public static void main(String[] args) {
        String painMessageXml = "<message>...your XML here...</message>"; // Input pain.001 XML
        
        // Parse the pain.001.001.12 message from XML into an MxPain00100112 object
        MxPain00100112 pain001 = MxPain00100112.parse(painMessageXml);
        if (pain001 == null || pain001.getCstmrCdtTrfInitn() == null) {
            System.out.println("Invalid pain.001.001.12 message");
            return;
        }

        // Create a new MxPacs00900107 object for the target message
        MxPacs00900107 pacs009 = new MxPacs00900107();
        FIToFICustomerCreditTransferV07 creditTransfer = new FIToFICustomerCreditTransferV07();
        pacs009.setFIToFICstmrCdtTrf(creditTransfer);

        // Map fields from pain.001.001.12 to pacs.009.001.07
        CustomerCreditTransferInitiationV12 cstmrCdtTrf = pain001.getCstmrCdtTrfInitn();

        // Map group header
        GroupHeader33 groupHeader = new GroupHeader33();
        groupHeader.setMsgId(cstmrCdtTrf.getGrpHdr().getMsgId());
        groupHeader.setCreDtTm(cstmrCdtTrf.getGrpHdr().getCreDtTm());
        creditTransfer.setGrpHdr(groupHeader);

        // Map payment transactions
        for (PaymentInstructionInformation3 payment : cstmrCdtTrf.getPmtInf()) {
            CreditTransferTransaction33 transfer = new CreditTransferTransaction33();

            // Set payment ID
            PaymentIdentification1 paymentId = new PaymentIdentification1();
            paymentId.setInstrId(payment.getPmtInfId());
            transfer.setPmtId(paymentId);

            // Map amount
            ActiveCurrencyAndAmount amount = new ActiveCurrencyAndAmount();
            amount.setCcy(payment.getCdtTrfTxInf().get(0).getInstdAmt().getCcy());
            amount.setValue(payment.getCdtTrfTxInf().get(0).getInstdAmt().getValue());
            transfer.setIntrBkSttlmAmt(amount);

            // Map debtor and creditor info
            transfer.setDbtr(payment.getDbtr());
            transfer.setDbtrAcct(payment.getDbtrAcct());
            transfer.setCdtr(payment.getCdtTrfTxInf().get(0).getCdtr());
            transfer.setCdtrAcct(payment.getCdtTrfTxInf().get(0).getCdtrAcct());

            // Add the transaction to the PACS message
            creditTransfer.getCdtTrfTxInf().add(transfer);
        }

        // Serialize the resulting MxPacs00900107 into XML
        String pacs009Xml = pacs009.message();
        System.out.println("Resulting pacs.009.001.07 XML:");
        System.out.println(pacs009Xml);
    }
}
