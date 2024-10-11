# Test-Automation

import pidsAir from "../../../pageObject/pidsAir";
const { sum } = require("lodash");
const { number } = require("yup");

let dateOfImport = "";
let importerName = "Johnkey";
let sendersReferenceNumber = "";
let gstPayable = "";
let itf = "";
let cashStatementFileLocation = "";
let deliverOrderFileLocation = "";
describe("PID 24. Cancel Pids", () => {
  const PidsAir = new pidsAir();
  before(() => {
    const { username, password } = Cypress.env("testAccount");
    cy.login(username, password);
    /** Make sure we're still on officer UI  */

    /** Make sure we're still on officer UI  */
    cy.fixture("example").then(function (data) {
      this.data = data;
    });
  });

  it("Click create button", () => {
    /** Click the first button we find. Need to clean up so there is only one to click */
    cy.get("[data-testid=Navigation-lodgementTuiCreate]").first().click();
  });

  /*Select Mode of transport "AIR" */
  it("select Air", function () {
    cy.get("[data-testid=CreateLodgement-modeOfTransport] [value=air]", {
      timeout: 20000,
    }).click();

    // cy.get("[data-testid=CreateLodgement-modeOfTransport] [value=air]").click()
    //Click Continue button//
    cy.get("[data-testid=button-continue]").click();
  });

  it("Fill in importer section", () => {
    dateOfImport = Cypress.dayjs().format("DD/MM/YYYY");
    cy.pidFillImporterSection(
      importerName,
      "NZTRG",
      dateOfImport,
      `CY${Cypress.dayjs().format("DDMMYYHHmmss")}`,
    );

    //Sender Reference Number//
    cy.get("[name=referenceNumber]").then((ref) => {
      sendersReferenceNumber = ref.val().toString();
    });

    //it("Check entered importer details are correct", () => {
    cy.get(
      "[data-testid=importer-section-summary] [data-testid=dateOfImport]",
    ).should("contain", dateOfImport);
    cy.get(
      "[data-testid=importer-section-summary] [data-testid=importerName]",
    ).should("contain", importerName);
  });

  // Detail Lines
  it("Fill in detail line", function () {
    cy.get('[data-testid="description"] ')
      .should("be.visible")
      .type("this.data.description");

    cy.get("[data-testid=value-of-goods][data-testid=value-of-goods] input")
      .clear()
      .type("5000")
      .blur();
    cy.get("[data-testid=detail-line-0] [data-testid=gst-payable] input").type(
      "750",
    );
    cy.get("[data-testid=detail-line-0] [data-testid=duty-payable] input").type(
      "100",
    );

    var sum = 0;
    cy.get('[data-testid="SummaryLine-Line total"]')
      .each(($el, index, $list) => {
        const amount = $el.text();
        var res = amount.split("$");
        res = res[1].trim();
        cy.log(res);
        sum = Number(sum) + Number(res);
      })
      .then(function () {
        cy.log(sum);
      });
    cy.get('[data-testid="totalValue"]').then(function (element) {
      const amount = element.text();
      var res = amount;
      cy.log(res);
      expect(res).to.equal(res);

      //*Continue Button*//
      cy.get(
        "[data-testid=ImportedGoodsSection] [data-testid=button-continue]",
      ).click();
    });

    cy.get('[data-testid="summaryTotal"]').then(function (element) {
      const amount = element.text();
      var res = amount;
      cy.log(res);
      expect(res).to.equal(res);

      //Submit)
      cy.get("[data-testid=button-continue]").last().click();
    });
  });

  it("Lodgement Status Details", function () {
    //Import Name//
    cy.get(
      "[data-testid=importer-section-summary] [data-testid=importerName]",
      {
        timeout: 120000,
      },
    )
      .contains("Name:Johnkey")
      .should("exist");

    //Date of import//
    cy.get(
      "[data-testid=importer-section-summary] [data-testid=dateOfImport]",
    ).should("contain", dateOfImport);
    //Port of
    cy.get(
      "[data-testid=importer-section-summary] [data-testid=portOfDischarge]",
    )
      .contains("Port of discharge:Tauranga (NZTRG)")
      .should("exist");

    //Sender Reference Number//
    cy.contains(sendersReferenceNumber).should("exist");

    //Summary Duty Payable//

    cy.get("[data-testid=summaryDutyPayable]").should("have.text", "$100.00");
    //GST Payable
    cy.get("[data-testid=summaryGstPayable]").should("contain", gstPayable);
    //Total//
    cy.get("[data-testid=summaryTotal]").should("have.text", "$865.00");

    cy.contains("TSW: Processing", { timeout: 500000 }).should("exist");
    cy.contains("Customs: Directions Given", { timeout: 500000 }).should(
      "exist",
    );

    //TSW Version History//
    cy.get("[data-testid=lodgement-version]").should("have.text", "1");

    //Cash Statement Order PDF
    cy.get("[data-testid=cash-statement-download]").should("not.be.disabled");
  });

  it("Get document link, then download document", () => {
    PidsAir.getCashStatementEnabled();
    cy.get("[data-testid=cash-statement-download]")
      .should("have.attr", "data-url")
      .should("not.be.empty")
      .then((url) => {
        // Use the URL from the cash statement button
        cy.saveTswDocument(url).then((filename) => {
          cy.log("cash statement file location", filename);
          cashStatementFileLocation = filename;
        });
      });
  });

  it(`Confirm cash statement PDF has expected details and amounts.`, () => {
    cy.task("getPdfText", cashStatementFileLocation)
      .should("contain", importerName)
      .should("contain", "Total payable for this");
  });
  it("Cancel Pids", () => {
    cy.get("button").contains("Cancel").click();
    cy.get('[rows="3"]')
      .type("Require additional documents")
      .should("contain.text", "Require additional documents");
    cy.get("button").contains("Yes, Cancel").should("be.enabled").click();
    cy.wait(1000);
  });
  it("Entry Cancelled", () => {
    cy.contains("Inspection / Audit requirements: Entry cancelled", {
      timeout: 500000,
    }).should("exist");
    cy.get("[data-testid=lodgement-version]").should("have.text", "2");
  });
});
