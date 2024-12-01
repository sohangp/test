# test

import com.prowidesoftware.swift.model.mx.MxPain00100112;
import com.prowidesoftware.swift.model.mx.MxPacs00800107;
import com.prowidesoftware.swift.model.mx.dic.*; // Provides access to ISO 20022 dictionary classes
import java.util.*;

public class PainToPacsConverter {

    public static void main(String[] args) {
        // 1. Parse a given pain.001 XML input (e.g., customer payment initiation message)
        String painXml = "<XML content for pain.001.001.12 here>"; // Replace with your actual XML input

        // Parse into MxPain00100112 object
        MxPain00100112 painMessage = MxPain00100112.parse(painXml);
        if (painMessage == null || painMessage.getCstmrCdtTrfInitn() == null) {
            System.out.println("Invalid pain.001.001.12 message");
            return;
        }

        // Extract the customer credit transfer initiation part
        CustomerCreditTransferInitiationV12 cstmrCdtTrfInitn = painMessage.getCstmrCdtTrfInitn();

        // 2. Create the target pacs.008.001.07 message
        MxPacs00800107 pacsMessage = new MxPacs00800107();

        FIToFICustomerCreditTransferV07 fiToFiCstmrCdtTrf = new FIToFICustomerCreditTransferV07();
        pacsMessage.setFIToFICstmrCdtTrf(fiToFiCstmrCdtTrf);

        // 3. Map the group header fields
        GroupHeader33 painGrpHdr = cstmrCdtTrfInitn.getGrpHdr();
        GroupHeader70 pacsGrpHdr = new GroupHeader70();

        pacsGrpHdr.setMsgId(painGrpHdr.getMsgId()); // Map message ID
        pacsGrpHdr.setCreDtTm(painGrpHdr.getCreDtTm()); // Map creation date and time

        // Additional mappings for group header (e.g., settlement information, initiating agent)
        pacsGrpHdr.setSttlmInf(new SettlementInstruction4()); // Example: Add settlement info if available
        fiToFiCstmrCdtTrf.setGrpHdr(pacsGrpHdr);

        // 4. Map each payment instruction in pain.001 to credit transfer transactions in pacs.008
        for (PaymentInstructionInformation3 pmtInf : cstmrCdtTrfInitn.getPmtInf()) {
            for (CreditTransferTransaction33 cdtTrfTx : pmtInf.getCdtTrfTxInf()) {
                CreditTransferTransaction39 pacsCdtTrfTx = new CreditTransferTransaction39();

                // Map payment information and identifiers
                pacsCdtTrfTx.setPmtId(new PaymentIdentification7());
                pacsCdtTrfTx.getPmtId().setInstrId(cdtTrfTx.getPmtId().getInstrId());
                pacsCdtTrfTx.getPmtId().setEndToEndId(cdtTrfTx.getPmtId().getEndToEndId());

                // Map amount and currency
                ActiveCurrencyAndAmount instdAmt = cdtTrfTx.getIntrBkSttlmAmt();
                pacsCdtTrfTx.setIntrBkSttlmAmt(new ActiveCurrencyAndAmount(instdAmt.getValue(), instdAmt.getCcy()));

                // Map debtor information
                pacsCdtTrfTx.setDbtr(pmtInf.getDbtr()); // Debtor
                pacsCdtTrfTx.setDbtrAcct(pmtInf.getDbtrAcct()); // Debtor Account
                pacsCdtTrfTx.setDbtrAgt(pmtInf.getDbtrAgt()); // Debtor Agent

                // Map creditor information
                pacsCdtTrfTx.setCdtr(cdtTrfTx.getCdtr()); // Creditor
                pacsCdtTrfTx.setCdtrAcct(cdtTrfTx.getCdtrAcct()); // Creditor Account
                pacsCdtTrfTx.setCdtrAgt(cdtTrfTx.getCdtrAgt()); // Creditor Agent

                // Add the mapped transaction to the PACS message
                fiToFiCstmrCdtTrf.getCdtTrfTxInf().add(pacsCdtTrfTx);
            }
        }

        // 5. Serialize the resulting pacs.008.001.07 message into XML
        String pacsXml = pacsMessage.message();
        System.out.println("Generated pacs.008.001.07 XML:");
        System.out.println(pacsXml);
    }
}
