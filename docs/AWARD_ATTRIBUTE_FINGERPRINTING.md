# Award Attribute Fingerprinting: A Robust Claim-Type Identification & Letter Routing System

## Table of Contents

- [The Problem Today](#the-problem-today)
- [The Award Attribute Fingerprint Model](#the-award-attribute-fingerprint-model)
  - [Core Concept](#core-concept)
  - [Fingerprint Dimensions](#fingerprint-dimensions)
  - [Mapping Award Types to Fingerprint Dimensions](#mapping-award-types-to-fingerprint-dimensions)
- [Proposed Implementation](#proposed-implementation)
  - [1. The Fingerprint Enum & Value Object](#1-the-fingerprint-enum--value-object)
  - [2. The Fingerprint Extractor](#2-the-fingerprint-extractor--builds-fingerprints-from-existing-data)
  - [3. The Letter Route Resolver](#3-the-letter-route-resolver--maps-fingerprints-to-letter-types)
  - [4. The Complete Fingerprint-to-Letter Mapping Table](#4-the-complete-fingerprint-to-letter-mapping-table)
- [How This Replaces Existing Logic](#how-this-replaces-existing-logic)
- [Key Design Decisions & Domain Rationale](#key-design-decisions--domain-rationale)
- [Dual Entitlement: Comp + Pension Overlap](#dual-entitlement-comp--pension-overlap)
  - [The Scenario](#the-scenario)
  - [Problems with the Current Approach](#problems-with-the-current-approach)
  - [How Fingerprinting Solves This](#how-fingerprinting-solves-this)
  - [Enhanced Implementation](#enhanced-implementation)
  - [Complete Comp + Pension Overlap Mapping Table](#complete-comp--pension-overlap-mapping-table)
  - [Why Booleans Cannot Solve This](#why-booleans-cannot-solve-this)
- [Next Steps](#next-steps)

---

## The Problem Today

The current system has **tightly coupled, scattered logic** for determining letter types. Key observations from the `vbms-awards` and `vbms-correspondence` repositories:

### 1. `RatingInformationDataConsumer.getLetterType()`

Uses a simple award type check (CPL/CPDS/CPDC/CPDP → PFS ADL, else → RADL):

```java
// RatingInformationDataConsumer.java
LetterTypeEnum getLetterType(String awardType) {
    boolean isPfsAdlEnabled = env.isPfsAdlEnabled();
    if (isPfsAdlEnabled && (
        AwardType.cplCode.equals(awardType)  ||
            AwardType.cpdsCode.equals(awardType) ||
            AwardType.cpdcCode.equals(awardType) ||
            AwardType.cpdpCode.equals(awardType))
    ) {
        return LetterTypeEnum.PFS_AUTOMATED_DECISION_LETTER;
    }
    return LetterTypeEnum.AUTOMATED_DECISION_LETTER;
}
```

### 2. `AwardsDataConsumer.validateClaimTypes()`

Uses EP code validation and out-of-scope lists with special bypass logic for PFS ADL claims:

```java
// AwardsDataConsumer.java
if (!request.isFeeAllocationNoticeLetter() && !request.isNrhlrDecision() && !request.isPfsAdlLetter()) {
    validateClaimTypes(award.getHAwardEvent(), award.getAwardType());
}
```

### 3. `displayaward.js`

Makes front-end routing decisions based on feature flags, eligibility booleans, and award type strings:

```javascript
// displayaward.js
if (!isPfsAdlEnabled || (!isEligibleForPfsAdl && awardType == 'CPL') || awardType == 'BUR') {
    displayCurrentAndProposedFormSave(awardsClaimsSize, successPage);
    return;
}
```

> **Summary:** This is fragile, hard to extend, and distributes routing decisions across Java services, JSPs, and JavaScript.

---

## The Award Attribute Fingerprint Model

### Core Concept

Every claim processed through the Awards system carries **inherent attributes** — properties that exist because of *what the claim is*, not because of any adjudicative decision. These attributes form a **composite fingerprint** that deterministically maps to a letter generation pipeline (PFS ADL vs. Comp RADL).

### Fingerprint Dimensions

Based on analysis of the codebase, there are **six fingerprint dimensions** derived from actual data structures in the system:

| #  | Dimension              | Source in Code                                      | Values                                                                 |
|----|------------------------|-----------------------------------------------------|------------------------------------------------------------------------|
| 1  | **Program Type**       | `AwardType` codes                                   | `PENSION`, `COMPENSATION`, `DIC`, `BURIAL`, `ACCRUED`, `SPECIAL`       |
| 2  | **Claimant Type**      | `AwardType` code suffix + `payeeType`               | `VETERAN`, `SPOUSE`, `CHILD`, `PARENT`                                 |
| 3  | **Benefit Category**   | `benefitTypeCd` (CPL/CPD) + Award Line Types        | `RECURRING`, `ONE_TIME`, `ACCRUED`                                     |
| 4  | **Service Connection** | Rating profile presence + Basic Eligibility Decisions | `SERVICE_CONNECTED`, `NON_SERVICE_CONNECTED`, `MIXED`                 |
| 5  | **Fiduciary Involvement** | Payee type code != "00"                          | `FIDUCIARY`, `NO_FIDUCIARY`                                            |
| 6  | **Claim Lane**         | EP code prefix (3-digit numeric)                    | `ORIGINAL`, `SUPPLEMENTAL`, `HLR`, `BVA`, `DEPENDENCY`, `SPECIAL`     |

### Mapping Award Types to Fingerprint Dimensions

The actual `AwardType` constants from the codebase:

```java
// AwardType.java
public static final String cplCode = "CPL";       // Compensation/Pension Live
public static final String mohCode = "MOH";       // Medal of Honor
public static final String burialCode = "BUR";    // Burial
public static final String accruedCode = "ACC";   // Accrued
public static final String cpdsCode = "CPDS";     // CPD Spouse (Death)
public static final String cpdcCode = "CPDC";     // CPD Child (Death)
public static final String cpdpCode = "CPDP";     // CPD Parent (Death)
public static final String caCode = "CA";         // Clothing Allowance
public static final String _306Veteran = "306V";  // Section 306 Veteran
public static final String oldLawVeteran = "OLV"; // Old Law Veteran
public static final String _306Spouse = "306S";   // Section 306 Spouse
public static final String _306Child = "306C";    // Section 306 Child
public static final String deathCompSpouse = "DCS"; // Death Comp Spouse
public static final String deathCompChild = "DCC";  // Death Comp Child
public static final String deathCompParent = "DCP"; // Death Comp Parent
public static final String _1312ASpouse = "1312S";  // 1312A Spouse
public static final String _1312AChild = "1312C";   // 1312A Child
public static final String _1312AParent = "1312P";  // 1312A Parent
```

---

## Proposed Implementation

### 1. The Fingerprint Enum & Value Object

```java
package gov.va.vba.award.fingerprint;

import java.util.Objects;

/**
 * Immutable value object representing the composite fingerprint of a claim.
 * Built from inherent claim attributes — NOT adjudicative decisions.
 */
public final class ClaimFingerprint {

    // ─── Dimension Enums ───

    public enum ProgramType {
        PENSION,          // Veterans Pension (IP, IDP, OLP, 306P, etc.)
        COMPENSATION,     // Service-connected disability compensation
        DIC,              // Dependency & Indemnity Compensation
        BURIAL,           // Burial benefits
        ACCRUED,          // Accrued benefits
        SPECIAL           // MOH, Clothing Allowance, CH18, etc.
    }

    public enum ClaimantType {
        VETERAN,
        SPOUSE,
        CHILD,
        PARENT
    }

    public enum BenefitCategory {
        RECURRING,        // Monthly ongoing payments
        ONE_TIME,         // Lump-sum (burial, accrued)
        ACCRUED           // Benefits accrued prior to death
    }

    public enum ServiceConnection {
        SERVICE_CONNECTED,
        NON_SERVICE_CONNECTED,
        MIXED              // CPL awards can contain both SC and NSC components
    }

    public enum FiduciaryStatus {
        FIDUCIARY,
        NO_FIDUCIARY
    }

    public enum ClaimLane {
        ORIGINAL,          // EP 010, 110, 020, 120, etc.
        SUPPLEMENTAL,      // EP 040 series
        HLR,               // EP 030 series (Higher Level Review)
        BVA,               // Board of Veterans' Appeals remands
        DEPENDENCY,        // EP 130, 600 series
        SPECIAL            // COLA, running awards, etc.
    }

    public enum DualEntitlementStatus {
        NONE,                          // Single-program claim, no overlap
        COMP_PENSION_ELECTION,         // Veteran has both SC comp and pension eligibility
        DIC_PENSION_ELECTION,          // Survivor has both DIC and death pension eligibility
        PENSION_DENIED_COMP_GREATER,   // Applied for pension, comp is greater benefit
        COMP_DENIED_PENSION_GREATER    // Had comp, pension is now greater benefit
    }

    // ─── Fields ───

    private final ProgramType programType;
    private final ClaimantType claimantType;
    private final BenefitCategory benefitCategory;
    private final ServiceConnection serviceConnection;
    private final FiduciaryStatus fiduciaryStatus;
    private final ClaimLane claimLane;
    private final DualEntitlementStatus dualEntitlement;

    private ClaimFingerprint(Builder builder) {
        this.programType = Objects.requireNonNull(builder.programType, "programType is required");
        this.claimantType = Objects.requireNonNull(builder.claimantType, "claimantType is required");
        this.benefitCategory = Objects.requireNonNull(builder.benefitCategory, "benefitCategory is required");
        this.serviceConnection = Objects.requireNonNull(builder.serviceConnection, "serviceConnection is required");
        this.fiduciaryStatus = Objects.requireNonNull(builder.fiduciaryStatus, "fiduciaryStatus is required");
        this.claimLane = Objects.requireNonNull(builder.claimLane, "claimLane is required");
        this.dualEntitlement = builder.dualEntitlement != null ? builder.dualEntitlement : DualEntitlementStatus.NONE;
    }

    // ─── Getters ───

    public ProgramType getProgramType() { return programType; }
    public ClaimantType getClaimantType() { return claimantType; }
    public BenefitCategory getBenefitCategory() { return benefitCategory; }
    public ServiceConnection getServiceConnection() { return serviceConnection; }
    public FiduciaryStatus getFiduciaryStatus() { return fiduciaryStatus; }
    public ClaimLane getClaimLane() { return claimLane; }
    public DualEntitlementStatus getDualEntitlement() { return dualEntitlement; }

    // ─── Identity ───

    /**
     * Returns a canonical string representation, e.g.:
     * "PENSION:VETERAN:RECURRING:NON_SERVICE_CONNECTED:NO_FIDUCIARY:ORIGINAL:NONE"
     */
    public String toCanonicalKey() {
        return String.join(":",
            programType.name(),
            claimantType.name(),
            benefitCategory.name(),
            serviceConnection.name(),
            fiduciaryStatus.name(),
            claimLane.name(),
            dualEntitlement.name()
        );
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ClaimFingerprint)) return false;
        ClaimFingerprint that = (ClaimFingerprint) o;
        return programType == that.programType
            && claimantType == that.claimantType
            && benefitCategory == that.benefitCategory
            && serviceConnection == that.serviceConnection
            && fiduciaryStatus == that.fiduciaryStatus
            && claimLane == that.claimLane
            && dualEntitlement == that.dualEntitlement;
    }

    @Override
    public int hashCode() {
        return Objects.hash(programType, claimantType, benefitCategory,
                           serviceConnection, fiduciaryStatus, claimLane, dualEntitlement);
    }

    @Override
    public String toString() {
        return "ClaimFingerprint[" + toCanonicalKey() + "]";
    }

    // ─── Builder ───

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private ProgramType programType;
        private ClaimantType claimantType;
        private BenefitCategory benefitCategory;
        private ServiceConnection serviceConnection;
        private FiduciaryStatus fiduciaryStatus;
        private ClaimLane claimLane;
        private DualEntitlementStatus dualEntitlement;

        public Builder programType(ProgramType val)             { this.programType = val; return this; }
        public Builder claimantType(ClaimantType val)           { this.claimantType = val; return this; }
        public Builder benefitCategory(BenefitCategory val)     { this.benefitCategory = val; return this; }
        public Builder serviceConnection(ServiceConnection val) { this.serviceConnection = val; return this; }
        public Builder fiduciaryStatus(FiduciaryStatus val)     { this.fiduciaryStatus = val; return this; }
        public Builder claimLane(ClaimLane val)                  { this.claimLane = val; return this; }
        public Builder dualEntitlement(DualEntitlementStatus val) { this.dualEntitlement = val; return this; }

        public ClaimFingerprint build() { return new ClaimFingerprint(this); }
    }
}
```

---

### 2. The Fingerprint Extractor — Builds Fingerprints from Existing Data

```java
package gov.va.vba.award.fingerprint;

import gov.va.vba.award.model.AwardType;
import gov.va.vba.award.fingerprint.ClaimFingerprint.*;
import org.apache.commons.lang.StringUtils;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

/**
 * Extracts a ClaimFingerprint from the inherent attributes of a claim.
 * This class centralizes all the "what kind of claim is this?" logic that
 * is currently scattered across the codebase.
 */
@Component
public class ClaimFingerprintExtractor {

    // ═══════════════════════════════════════════════════════════════
    // PENSION Award Types (PFS territory)
    // These are non-service-connected, needs-based benefits
    // administered by Pension & Fiduciary Services.
    //
    // Award Line Types: IP (Improved Pension), IDP (Improved Death
    // Pension), OLP (Old Law Pension), 306P (Section 306 Pension),
    // 306DP (Section 306 Death Pension)
    // ═══════════════════════════════════════════════════════════════

    /** Live veteran pension award types */
    private static final Set<String> PENSION_VETERAN_TYPES = new HashSet<>(Arrays.asList(
        AwardType.cplCode      // CPL — when benefitType is "CP" with pension award lines (IP, OLP, 306P)
    ));

    /** Section 306 and Old Law pension types — inherently pension, always PFS */
    private static final Set<String> PENSION_LEGACY_VETERAN_TYPES = new HashSet<>(Arrays.asList(
        AwardType._306Veteran,   // 306V
        AwardType.oldLawVeteran  // OLV
    ));

    /** Death pension types — surviving dependents of non-SC veterans */
    private static final Set<String> DEATH_PENSION_SPOUSE_TYPES = new HashSet<>(Arrays.asList(
        AwardType.cpdsCode,      // CPDS — CPD Spouse (DIC/Death Pension Spouse)
        AwardType._306Spouse,    // 306S
        AwardType.oldLawSpouse   // OLS
    ));

    private static final Set<String> DEATH_PENSION_CHILD_TYPES = new HashSet<>(Arrays.asList(
        AwardType.cpdcCode,      // CPDC — CPD Child
        AwardType._306Child,     // 306C
        AwardType.oldLawChild    // OLC
    ));

    private static final Set<String> DEATH_PENSION_PARENT_TYPES = new HashSet<>(Arrays.asList(
        AwardType.cpdpCode       // CPDP — CPD Parent
    ));

    // ═══════════════════════════════════════════════════════════════
    // COMPENSATION Award Types (Comp Services / RADL territory)
    // ═══════════════════════════════════════════════════════════════

    /** DIC types — service-connected death benefits (Comp Service) */
    private static final Set<String> DIC_SPOUSE_TYPES = new HashSet<>(Arrays.asList(
        AwardType.deathCompSpouse,  // DCS
        AwardType._1312ASpouse      // 1312S
    ));

    private static final Set<String> DIC_CHILD_TYPES = new HashSet<>(Arrays.asList(
        AwardType.deathCompChild,   // DCC
        AwardType._1312AChild       // 1312C
    ));

    private static final Set<String> DIC_PARENT_TYPES = new HashSet<>(Arrays.asList(
        AwardType.deathCompParent,  // DCP
        AwardType._1312AParent      // 1312P
    ));

    // ═══════════════════════════════════════════════════════════════
    // EP Code Classification
    // ═══════════════════════════════════════════════════════════════

    private static final Set<String> HLR_EP_PREFIXES = new HashSet<>(Arrays.asList("030"));
    private static final Set<String> SUPPLEMENTAL_EP_PREFIXES = new HashSet<>(Arrays.asList("040", "930"));
    private static final Set<String> ORIGINAL_COMP_EP_PREFIXES = new HashSet<>(Arrays.asList(
        "010", "110", "020", "120"
    ));
    private static final Set<String> PENSION_EP_PREFIXES = new HashSet<>(Arrays.asList(
        "150", "180"
    ));
    private static final Set<String> DEPENDENCY_EP_PREFIXES = new HashSet<>(Arrays.asList(
        "130", "600"
    ));

    // Pension award line type codes (from AccruedDecisionsController)
    private static final Set<String> PENSION_AWARD_LINE_TYPES = new HashSet<>(Arrays.asList(
        "IP", "IDP", "OLDP", "OLP", "306P", "306DP", "1312A"
    ));

    // ═══════════════════════════════════════════════════════════════
    // EXTRACTION
    // ═══════════════════════════════════════════════════════════════

    /**
     * Builds a ClaimFingerprint from the attributes inherent to the claim.
     *
     * @param awardType       The AwardType code (CPL, CPDS, BUR, etc.)
     * @param benefitTypeCd   The benefit type code (CP, CPL, CPD, etc.)
     * @param endProductType  The full EP claim type string (e.g., "110LCOMP")
     * @param payeeTypeCode   The payee type code ("00" = veteran, others = fiduciary)
     * @param awardLineTypes  The set of award line type codes on this award (IP, DC, etc.)
     * @param hasRatingProfile Whether a rating profile exists for this claim
     * @return A fully-constructed ClaimFingerprint
     */
    public ClaimFingerprint extract(
            String awardType,
            String benefitTypeCd,
            String endProductType,
            String payeeTypeCode,
            Set<String> awardLineTypes,
            boolean hasRatingProfile) {

        return ClaimFingerprint.builder()
            .programType(resolveProgramType(awardType, benefitTypeCd, endProductType, awardLineTypes))
            .claimantType(resolveClaimantType(awardType))
            .benefitCategory(resolveBenefitCategory(awardType, awardLineTypes))
            .serviceConnection(resolveServiceConnection(awardType, awardLineTypes, hasRatingProfile))
            .fiduciaryStatus(resolveFiduciaryStatus(payeeTypeCode))
            .claimLane(resolveClaimLane(endProductType))
            .build();
    }

    // ─── Dimension Resolvers ───

    private ProgramType resolveProgramType(
            String awardType, String benefitTypeCd,
            String endProductType, Set<String> awardLineTypes) {

        // Explicit non-ambiguous types first
        if (AwardType.burialCode.equals(awardType)) return ProgramType.BURIAL;
        if (AwardType.accruedCode.equals(awardType)) return ProgramType.ACCRUED;
        if (AwardType.mohCode.equals(awardType)) return ProgramType.SPECIAL;
        if (AwardType.caCode.equals(awardType)) return ProgramType.SPECIAL;
        if (AwardType.ch18SpinaBifida.equals(awardType)
            || AwardType.ch18BirthDefects.equals(awardType)) {
            return ProgramType.SPECIAL;
        }

        // Legacy pension types — always PFS
        if (PENSION_LEGACY_VETERAN_TYPES.contains(awardType)) return ProgramType.PENSION;
        if (DEATH_PENSION_SPOUSE_TYPES.contains(awardType)
            && containsPensionAwardLines(awardLineTypes)) {
            return ProgramType.PENSION;
        }
        if (DEATH_PENSION_CHILD_TYPES.contains(awardType)
            && containsPensionAwardLines(awardLineTypes)) {
            return ProgramType.PENSION;
        }

        // DIC types — always Compensation Service
        if (DIC_SPOUSE_TYPES.contains(awardType)) return ProgramType.DIC;
        if (DIC_CHILD_TYPES.contains(awardType)) return ProgramType.DIC;
        if (DIC_PARENT_TYPES.contains(awardType)) return ProgramType.DIC;

        // CPL — the ambiguous one. Resolve via EP code and award lines.
        if (AwardType.cplCode.equals(awardType)) {
            String epPrefix = extractEpPrefix(endProductType);
            if (PENSION_EP_PREFIXES.contains(epPrefix)) return ProgramType.PENSION;
            if (containsOnlyPensionAwardLines(awardLineTypes)) return ProgramType.PENSION;
            return ProgramType.COMPENSATION;
        }

        // CPDS/CPDC/CPDP — Death types
        if (AwardType.cpdsCode.equals(awardType)
            || AwardType.cpdcCode.equals(awardType)
            || AwardType.cpdpCode.equals(awardType)) {
            if (awardLineTypes != null
                && (awardLineTypes.contains("DIC") || awardLineTypes.contains("DICR")
                    || awardLineTypes.contains("DICP") || awardLineTypes.contains("DC"))) {
                return ProgramType.DIC;
            }
            if (containsPensionAwardLines(awardLineTypes)) {
                return ProgramType.PENSION;
            }
            return ProgramType.DIC;
        }

        return ProgramType.COMPENSATION;
    }

    private ClaimantType resolveClaimantType(String awardType) {
        if (Arrays.asList(AwardType.cpdsCode, AwardType._306Spouse, AwardType.oldLawSpouse,
                AwardType.deathCompSpouse, AwardType._1312ASpouse, AwardType.repsCode)
                .contains(awardType)) {
            return ClaimantType.SPOUSE;
        }
        if (Arrays.asList(AwardType.cpdcCode, AwardType._306Child, AwardType.oldLawChild,
                AwardType.deathCompChild, AwardType._1312AChild)
                .contains(awardType)) {
            return ClaimantType.CHILD;
        }
        if (Arrays.asList(AwardType.cpdpCode, AwardType.deathCompParent, AwardType._1312AParent)
                .contains(awardType)) {
            return ClaimantType.PARENT;
        }
        return ClaimantType.VETERAN;
    }

    private BenefitCategory resolveBenefitCategory(String awardType, Set<String> awardLineTypes) {
        if (AwardType.accruedCode.equals(awardType)) return BenefitCategory.ACCRUED;
        if (AwardType.burialCode.equals(awardType)) return BenefitCategory.ONE_TIME;
        return BenefitCategory.RECURRING;
    }

    private ServiceConnection resolveServiceConnection(
            String awardType, Set<String> awardLineTypes, boolean hasRatingProfile) {

        if (PENSION_LEGACY_VETERAN_TYPES.contains(awardType))
            return ServiceConnection.NON_SERVICE_CONNECTED;
        if (DIC_SPOUSE_TYPES.contains(awardType) || DIC_CHILD_TYPES.contains(awardType)
            || DIC_PARENT_TYPES.contains(awardType)) {
            return ServiceConnection.SERVICE_CONNECTED;
        }
        if (AwardType.cplCode.equals(awardType)) {
            boolean hasPensionLines = containsPensionAwardLines(awardLineTypes);
            boolean hasCompLines = awardLineTypes != null && awardLineTypes.stream()
                .anyMatch(lt -> !PENSION_AWARD_LINE_TYPES.contains(lt));
            if (hasPensionLines && hasCompLines) return ServiceConnection.MIXED;
            if (hasPensionLines) return ServiceConnection.NON_SERVICE_CONNECTED;
            return ServiceConnection.SERVICE_CONNECTED;
        }
        if (hasRatingProfile) return ServiceConnection.SERVICE_CONNECTED;
        return ServiceConnection.NON_SERVICE_CONNECTED;
    }

    private FiduciaryStatus resolveFiduciaryStatus(String payeeTypeCode) {
        if (StringUtils.isBlank(payeeTypeCode) || "00".equals(payeeTypeCode)) {
            return FiduciaryStatus.NO_FIDUCIARY;
        }
        return FiduciaryStatus.FIDUCIARY;
    }

    private ClaimLane resolveClaimLane(String endProductType) {
        String prefix = extractEpPrefix(endProductType);
        if (prefix == null) return ClaimLane.ORIGINAL;
        if (HLR_EP_PREFIXES.contains(prefix)) return ClaimLane.HLR;
        if (SUPPLEMENTAL_EP_PREFIXES.contains(prefix)) return ClaimLane.SUPPLEMENTAL;
        if (DEPENDENCY_EP_PREFIXES.contains(prefix)) return ClaimLane.DEPENDENCY;
        return ClaimLane.ORIGINAL;
    }

    // ─── Helpers ───

    private String extractEpPrefix(String endProductType) {
        if (StringUtils.isBlank(endProductType) || endProductType.length() < 3) return null;
        return endProductType.substring(0, 3);
    }

    private boolean containsPensionAwardLines(Set<String> awardLineTypes) {
        if (awardLineTypes == null || awardLineTypes.isEmpty()) return false;
        return awardLineTypes.stream().anyMatch(PENSION_AWARD_LINE_TYPES::contains);
    }

    private boolean containsOnlyPensionAwardLines(Set<String> awardLineTypes) {
        if (awardLineTypes == null || awardLineTypes.isEmpty()) return false;
        return PENSION_AWARD_LINE_TYPES.containsAll(awardLineTypes);
    }
}
```

---

### 3. The Letter Route Resolver — Maps Fingerprints to Letter Types

```java
package gov.va.vba.award.fingerprint;

import gov.va.vba.award.fingerprint.ClaimFingerprint.*;
import org.springframework.stereotype.Component;

/**
 * Maps ClaimFingerprints to letter generation pipelines.
 *
 * LETTER TYPES:
 * - PFS_ADL:  PFS Award Decision Letter (Pension & Fiduciary Services)
 * - COMP_RADL: Redesigned Automated Decision Letter (Compensation Services)
 * - BURIAL_LETTER: Burial-specific letter template
 * - NRHLR_DECISION: Non-Rating Higher Level Review Decision Letter
 * - NO_LETTER: Out-of-scope claim types (CA, MOH, CH18, etc.)
 */
@Component
public class LetterRouteResolver {

    public enum LetterRoute {
        PFS_ADL("PFS Automated Decision Letter"),
        COMP_RADL("Compensation Redesigned Automated Decision Letter"),
        BURIAL_LETTER("Burial Compensation Letter"),
        NRHLR_DECISION("Non-Rating Higher Level Review Decision Letter"),
        FEE_ALLOCATION_NOTICE("Fee Allocation Notice Letter"),
        NO_LETTER("No automated letter generated"),

        // ─── Dual Entitlement Composite Routes ───
        COMP_RADL_WITH_PENSION_DENIAL(
            "RADL with embedded pension denial — comp is the greater benefit"),
        PFS_ADL_WITH_COMP_RATING(
            "PFS ADL with embedded comp rating info — pension is the greater benefit"),
        COMP_RADL_WITH_ELECTION_NOTICE(
            "RADL with benefit election notice — Veteran must choose");

        private final String description;
        LetterRoute(String description) { this.description = description; }
        public String getDescription() { return description; }

        /** Returns true if this route requires PFS letter content sections */
        public boolean requiresPfsContent() {
            return this == PFS_ADL
                || this == PFS_ADL_WITH_COMP_RATING
                || this == COMP_RADL_WITH_PENSION_DENIAL;
        }

        /** Returns true if this route requires Comp/RADL content sections */
        public boolean requiresCompContent() {
            return this == COMP_RADL
                || this == COMP_RADL_WITH_PENSION_DENIAL
                || this == COMP_RADL_WITH_ELECTION_NOTICE
                || this == PFS_ADL_WITH_COMP_RATING;
        }
    }

    /**
     * Resolves the letter route for a given claim fingerprint.
     *
     * ROUTING RULES (ordered by specificity):
     *
     * RULE 1: SPECIAL programs → NO_LETTER
     * RULE 2: BURIAL → BURIAL_LETTER
     * RULE 3: HLR claim lane → NRHLR_DECISION
     * RULE 4: Dual entitlement → composite route (see below)
     * RULE 5: PENSION program → PFS_ADL
     * RULE 6: NON_SERVICE_CONNECTED → PFS_ADL
     * RULE 7: DIC program → COMP_RADL
     * RULE 8: COMPENSATION → COMP_RADL
     * RULE 9: MIXED service connection → COMP_RADL
     * RULE 10: ACCRUED → route based on underlying program
     */
    public LetterRoute resolve(ClaimFingerprint fingerprint) {

        // Rule 1: Special programs — no automated letter
        if (fingerprint.getProgramType() == ProgramType.SPECIAL) {
            return LetterRoute.NO_LETTER;
        }

        // Rule 2: Burial — dedicated template
        if (fingerprint.getProgramType() == ProgramType.BURIAL) {
            return LetterRoute.BURIAL_LETTER;
        }

        // Rule 3: HLR lane — Non-Rating HLR Decision Letter
        if (fingerprint.getClaimLane() == ClaimLane.HLR) {
            return LetterRoute.NRHLR_DECISION;
        }

        // Rule 4: Dual entitlement — composite letter needed
        if (fingerprint.getDualEntitlement() != DualEntitlementStatus.NONE) {
            return resolveDualEntitlementRoute(fingerprint);
        }

        // Rule 5: Pension program — always PFS ADL
        if (fingerprint.getProgramType() == ProgramType.PENSION) {
            return LetterRoute.PFS_ADL;
        }

        // Rule 6: Non-service-connected claims — PFS ADL
        if (fingerprint.getServiceConnection() == ServiceConnection.NON_SERVICE_CONNECTED) {
            return LetterRoute.PFS_ADL;
        }

        // Rule 7: DIC — Compensation RADL (service-connected death)
        if (fingerprint.getProgramType() == ProgramType.DIC) {
            return LetterRoute.COMP_RADL;
        }

        // Rule 8: Compensation — RADL
        if (fingerprint.getProgramType() == ProgramType.COMPENSATION) {
            return LetterRoute.COMP_RADL;
        }

        // Rule 9: Mixed service connection — RADL takes primary ownership
        if (fingerprint.getServiceConnection() == ServiceConnection.MIXED) {
            return LetterRoute.COMP_RADL;
        }

        // Rule 10: Accrued — route based on service connection
        if (fingerprint.getProgramType() == ProgramType.ACCRUED) {
            if (fingerprint.getServiceConnection() == ServiceConnection.SERVICE_CONNECTED) {
                return LetterRoute.COMP_RADL;
            }
            return LetterRoute.PFS_ADL;
        }

        // Fallback
        return LetterRoute.COMP_RADL;
    }

    /**
     * Dual entitlement routing.
     *
     * SCENARIO A: Comp is the greater benefit
     *   Primary letter: COMP_RADL
     *   Embedded PFS content: pension denial, pension rate, election rights
     *
     * SCENARIO B: Pension is the greater benefit
     *   Primary letter: PFS_ADL
     *   Embedded Comp content: SC rating decisions, combined evaluation
     *   Right to revert to comp if circumstances change
     *
     * SCENARIO C: Election pending
     *   Primary letter: COMP_RADL (with election notice)
     *   Veteran must be informed of both rates and asked to elect
     */
    private LetterRoute resolveDualEntitlementRoute(ClaimFingerprint fingerprint) {
        switch (fingerprint.getDualEntitlement()) {
            case PENSION_DENIED_COMP_GREATER:
                return LetterRoute.COMP_RADL_WITH_PENSION_DENIAL;

            case COMP_DENIED_PENSION_GREATER:
                return LetterRoute.PFS_ADL_WITH_COMP_RATING;

            case COMP_PENSION_ELECTION:
                return LetterRoute.COMP_RADL_WITH_ELECTION_NOTICE;

            case DIC_PENSION_ELECTION:
                return LetterRoute.COMP_RADL_WITH_PENSION_DENIAL;

            default:
                return LetterRoute.COMP_RADL;
        }
    }

    /**
     * Returns metadata about the routing decision for audit/logging.
     */
    public LetterRouteDecision resolveWithMetadata(ClaimFingerprint fingerprint) {
        LetterRoute route = resolve(fingerprint);
        return new LetterRouteDecision(fingerprint, route);
    }

    /**
     * Audit-friendly record of how the routing decision was made.
     */
    public static class LetterRouteDecision {
        private final ClaimFingerprint fingerprint;
        private final LetterRoute route;
        private final long resolvedAt;

        LetterRouteDecision(ClaimFingerprint fingerprint, LetterRoute route) {
            this.fingerprint = fingerprint;
            this.route = route;
            this.resolvedAt = System.currentTimeMillis();
        }

        public ClaimFingerprint getFingerprint() { return fingerprint; }
        public LetterRoute getRoute() { return route; }
        public long getResolvedAt() { return resolvedAt; }

        @Override
        public String toString() {
            return String.format("LetterRouteDecision[%s -> %s (%s)]",
                fingerprint.toCanonicalKey(), route.name(), route.getDescription());
        }
    }
}
```

---

### 4. The Complete Fingerprint-to-Letter Mapping Table

| Fingerprint | Source Award Types | Program | Claimant | Benefit | SC? | Fiduciary | Lane | → Letter |
|---|---|---|---|---|---|---|---|---|
| `PENSION:VETERAN:RECURRING:NSC:NO_FID:ORIG` | `CPL` (w/ pension EP prefixes or pension-only award lines), `306V`, `OLV` | Pension | Veteran | Recurring | No | No | Original | **PFS ADL** |
| `PENSION:VETERAN:RECURRING:NSC:FID:ORIG` | `CPL` (w/ pension EP prefixes or pension-only award lines), `306V`, `OLV` | Pension | Veteran | Recurring | No | Yes | Original | **PFS ADL** (Fiduciary variant) |
| `PENSION:SPOUSE:RECURRING:NSC:NO_FID:ORIG` | `CPDS` (w/ IDP/306DP lines), `306S`, `OLS` | Death Pension | Spouse | Recurring | No | No | Original | **PFS ADL** |
| `PENSION:CHILD:RECURRING:NSC:NO_FID:ORIG` | `CPDC` (w/ IDP/306DP lines), `306C`, `OLC` | Death Pension | Child | Recurring | No | No | Original | **PFS ADL** |
| `PENSION:PARENT:RECURRING:NSC:NO_FID:ORIG` | `CPDP` (w/ IDP/306DP lines) | Death Pension | Parent | Recurring | No | No | Original | **PFS ADL** |
| `COMP:VETERAN:RECURRING:SC:NO_FID:ORIG` | `CPL` (w/ comp EP prefixes or comp award lines) | Comp | Veteran | Recurring | Yes | No | Original | **COMP RADL** |
| `COMP:VETERAN:RECURRING:SC:NO_FID:SUPP` | `CPL` (w/ comp EP prefixes or comp award lines) | Comp | Veteran | Recurring | Yes | No | Supplemental | **COMP RADL** |
| `COMP:VETERAN:RECURRING:SC:FID:ORIG` | `CPL` (w/ comp EP prefixes or comp award lines) | Comp | Veteran | Recurring | Yes | Yes | Original | **COMP RADL** (Fiduciary) |
| `COMP:VETERAN:RECURRING:MIXED:NO_FID:ORIG` | `CPL` (w/ both comp and pension lines) | Comp+Pension | Veteran | Recurring | Mixed | No | Original | **COMP RADL** (w/ pension sections) |
| `DIC:SPOUSE:RECURRING:SC:NO_FID:ORIG` | `CPDS` (w/ DIC/DICR/DICP/DC lines), `DCS`, `1312S` | DIC | Spouse | Recurring | Yes | No | Original | **COMP RADL** |
| `DIC:CHILD:RECURRING:SC:NO_FID:ORIG` | `CPDC` (w/ DIC/DICR/DICP/DC lines), `DCC`, `1312C` | DIC | Child | Recurring | Yes | No | Original | **COMP RADL** |
| `DIC:PARENT:RECURRING:SC:NO_FID:ORIG` | `CPDP` (w/ DIC/DICR/DICP/DC lines), `DCP`, `1312P` | DIC | Parent | Recurring | Yes | No | Original | **COMP RADL** |
| `BURIAL:VETERAN:ONE_TIME:*:*:ORIG` | `BUR` | Burial | Any | One-time | Any | Any | Original | **BURIAL LETTER** |
| `SPECIAL:VETERAN:*:*:*:*` | `MOH`, `CA`, `CH18` | MOH/CA/CH18 | Any | Any | Any | Any | Any | **NO LETTER** |
| `*:*:*:*:*:HLR` | Any (HLR lane) | Any | Any | Any | Any | Any | HLR | **NRHLR DECISION** |

---

## How This Replaces Existing Logic

### Before (scattered boolean checks)

```
AuthorizeAwardLogic → checks isPfsAdlEnabled + isEligibleForPfsAdl boolean
  ↓
AwardCompensationLetterConverter → checks isPfsAdlLetter from holder + isPfsAdlEnabled
  ↓
RatingInformationDataConsumer → checks awardType ∈ {CPL,CPDS,CPDC,CPDP}
  ↓
AwardsDataConsumer → skips validateClaimTypes if isPfsAdlLetter
  ↓
displayaward.js → checks !isPfsAdlEnabled || (!isEligibleForPfsAdl && awardType=='CPL') || awardType=='BUR'
```

### After (centralized fingerprint)

```
ClaimFingerprintExtractor.extract(awardType, benefitType, epCode, payeeType, awardLines, hasRating)
  ↓
LetterRouteResolver.resolve(fingerprint) → PFS_ADL | COMP_RADL | BURIAL | NRHLR | NO_LETTER
  ↓
All downstream consumers read the route, not the raw attributes.
```

### Specific Integration Point

Replace `RatingInformationDataConsumer.getLetterType()`:

```java
// BEFORE:
LetterTypeEnum getLetterType(String awardType) {
    boolean isPfsAdlEnabled = env.isPfsAdlEnabled();
    if (isPfsAdlEnabled && (AwardType.cplCode.equals(awardType) || ...)) {
        return LetterTypeEnum.PFS_AUTOMATED_DECISION_LETTER;
    }
    return LetterTypeEnum.AUTOMATED_DECISION_LETTER;
}

// AFTER:
LetterTypeEnum getLetterType(ClaimFingerprint fingerprint) {
    LetterRoute route = letterRouteResolver.resolve(fingerprint);
    switch (route) {
        case PFS_ADL:         return LetterTypeEnum.PFS_AUTOMATED_DECISION_LETTER;
        case COMP_RADL:       return LetterTypeEnum.AUTOMATED_DECISION_LETTER;
        case BURIAL_LETTER:   return LetterTypeEnum.BURIAL_LETTER;
        case NRHLR_DECISION:  return LetterTypeEnum.NON_RATING_HLR_DECISION_LETTER;
        default:              return null; // NO_LETTER
    }
}
```

---

## Key Design Decisions & Domain Rationale

### Why CPL Is the "Ambiguous" Award Type

CPL ("Compensation/Pension Live") is used for **both** compensation and pension claims involving a living veteran. The system disambiguates using:

- **EP code prefix**: `150`/`180` = Pension-specific EPs
- **Benefit type code**: "CP" can mean either
- **Award line types**: `IP`/`OLP`/`306P` = pension; SC disability lines = compensation

This is why single-attribute routing fails — CPL alone tells you nothing.

### Why CPDS/CPDC/CPDP Span Both Services

These "CPD" (Compensation/Pension Death) types cover **both** DIC (Compensation Service) and Death Pension (PFS). A surviving spouse could receive either:

- **DIC** (38 USC §1310) — if veteran's death was service-connected → **RADL**
- **Death Pension** (38 USC §1541) — if veteran had wartime service and surviving spouse has low income → **PFS ADL**

The award line types (`DIC`/`DICR`/`DICP` vs. `IDP`/`306DP`) are the differentiator.

### Why Fiduciary Is a Dimension

PFS manages fiduciary appointments. When a fiduciary is involved (payee type ≠ "00"), the letter content changes (different address routing, different legal notices). This is inherent to the claim — a beneficiary either has an appointed fiduciary or doesn't.

### Edge Case: CPDS/CPDC/CPDP with Both DIC and Pension Award Lines

When a CPDS/CPDC/CPDP claim has **both** DIC lines (DIC, DICR, DICP, DC) **and** pension lines (IDP, 306DP, etc.) on the same award, the current extractor resolves to `ProgramType.DIC` because DIC lines are checked first in the `resolveProgramType()` method (lines 472-475 in the proposed implementation).

**Current Behavior:**

```java
// CPDS/CPDC/CPDP — Death types
if (AwardType.cpdsCode.equals(awardType)
    || AwardType.cpdcCode.equals(awardType)
    || AwardType.cpdpCode.equals(awardType)) {
    if (awardLineTypes != null
        && (awardLineTypes.contains("DIC") || awardLineTypes.contains("DICR")
            || awardLineTypes.contains("DICP") || awardLineTypes.contains("DC"))) {
        return ProgramType.DIC;  // ← DIC checked first
    }
    if (containsPensionAwardLines(awardLineTypes)) {
        return ProgramType.PENSION;  // ← Pension checked second
    }
    return ProgramType.DIC;  // ← Default to DIC if no lines match
}
```

**Implications:**

- The pension lines are **silently ignored** for routing purposes when DIC lines are present
- The claim routes to **COMP RADL**, not PFS ADL
- This is a deliberate precedence decision: **DIC (service-connected death) takes priority over Death Pension (non-service-connected)**

**Business Rationale:**

DIC is a higher-value, service-connected benefit that is mutually exclusive with Death Pension under 38 USC §5304(a)(1). When both types of award lines appear on the same CPDS/CPDC/CPDP claim, it typically represents one of these scenarios:

1. **Award conversion**: A beneficiary previously receiving Death Pension is now being granted DIC retroactively (e.g., due to a new service-connection determination for the veteran's cause of death)
2. **Dual claim adjudication**: Both DIC and Death Pension were evaluated, and DIC was granted (making the pension lines moot)
3. **Historical remnants**: Legacy data where both line types exist due to system migrations or corrections

In all cases, routing to COMP RADL (via `ProgramType.DIC`) is the correct behavior because:
- DIC requires rating-related content (service connection determinations, dependency status)
- The Compensation Service handles DIC claims, not PFS
- If pension was denied in favor of DIC, that decision needs to be explained in RADL content

**Open Question:**

Should this instead resolve to `ProgramType.MIXED` (like the CPL dual entitlement case) or generate two separate letters? This is flagged for further analysis in **Next Steps #6** (overlap handling). The current implementation prioritizes DIC routing as the safer default, ensuring service-connected death benefits receive appropriate rating content.

---

## Dual Entitlement: Comp + Pension Overlap

### The Scenario

A Veteran is already receiving **service-connected disability compensation** (e.g., rated at 30%) and files a new claim for **Veterans Pension** (non-service-connected, needs-based). This happens when:

- The Veteran has **wartime service** and is age 65+ or permanently/totally disabled from non-SC conditions
- The Veteran's **income is low enough** to qualify for pension
- The Veteran may believe the pension rate is **higher than their current comp rate** (especially if they have high unreimbursed medical expenses that reduce countable income)

Under VA rules (38 USC §5304), a Veteran **cannot receive both compensation and pension simultaneously** — they must elect the greater benefit. This is the "**dual entitlement / election**" scenario.

### Problems with the Current Approach

#### Problem 1: CPL Award Type Is Shared

Both the existing compensation award and the new pension claim live under **`AwardType.cplCode` ("CPL")**. The system can't distinguish them by award type alone.

The current `RatingInformationDataConsumer.getLetterType()` sees "CPL" and returns `PFS_AUTOMATED_DECISION_LETTER` — but this Veteran's award has **both SC compensation AND a pension application**. The letter needs to address:
- The existing compensation rating decisions (RADL content)
- The pension eligibility determination (PFS ADL content)
- The dual entitlement election (unique to this overlap)

A single boolean (`isPfsAdlLetter = true/false`) can't capture this — it's **both**.

#### Problem 2: The `isEligibleForPfsAdl` Boolean Is Binary

The front-end dialog currently offers an either/or choice. When `isEligibleForPfsAdl` is `true` and `awardType == 'CPL'`, the dialog shows the PFS ADL option. But this Veteran also has SC disability ratings that need to appear on the letter. If the user selects PFS ADL, the compensation rating information may be omitted. If they select RADL, the pension decision content may be omitted.

#### Problem 3: Award Lines Contain Both Program Types Simultaneously

In the comp-to-pension scenario, the proposed award event may contain:
- **Compensation award lines** (SC disability, combined rating of 30%)
- **Pension award line** (`"IP"` — Improved Pension) if pension is the greater benefit
- **Or** a pension **denial** if compensation remains greater

The `validateClaimTypes()` method in `AwardsDataConsumer` currently bypasses validation when `isPfsAdlLetter` is true — but that bypass may incorrectly skip validation on the compensation EP codes that are legitimately in scope.

#### Problem 4: Pension Denial on a Comp Award Still Needs PFS Content

Even if pension is **denied** (because compensation is the greater benefit), the letter must explain:
- Why pension was denied
- What the pension rate would have been
- The Veteran's right to elect pension in the future if circumstances change

This is PFS content that belongs on what would otherwise be a Comp RADL.

#### Problem 5: Basic Eligibility Decisions Span Both Services

The `BasicEligibilityDecisionController` filters PFS-specific eligibility decisions (DCCNM, DD, DRM) when PFS ADL is disabled. In the comp+pension scenario, the award may have **both** compensation eligibility decisions (Eligible Beneficiary for comp) **and** pension eligibility decisions (Pension Grant) simultaneously. The current filtering logic doesn't account for this overlap.

### How Fingerprinting Solves This

#### The Key Insight: The `MIXED` Service Connection Value

The fingerprint model already has the escape valve for this exact scenario. When the extractor sees a CPL award with **both** compensation and pension award lines, it resolves to:

```
COMPENSATION:VETERAN:RECURRING:MIXED:NO_FIDUCIARY:ORIGINAL
                                ^^^^^
                          This is the key
```

The `MIXED` service connection value explicitly represents the dual-program state.

#### The 7th Dimension: `DualEntitlementStatus`

To fully handle the comp+pension overlap, the fingerprint model includes a **7th dimension** — `DualEntitlementStatus` — which classifies the specific flavor of dual entitlement:

| Status | Meaning |
|---|---|
| `NONE` | Single-program claim, no overlap |
| `COMP_PENSION_ELECTION` | Veteran has both SC comp and pension eligibility — election needed |
| `DIC_PENSION_ELECTION` | Survivor has both DIC and death pension eligibility |
| `PENSION_DENIED_COMP_GREATER` | Applied for pension, comp is greater benefit |
| `COMP_DENIED_PENSION_GREATER` | Had comp, pension is now the greater benefit |

### Enhanced Implementation

#### Dual Entitlement Extractor

```java
/**
 * DUAL ENTITLEMENT resolution.
 *
 * Detects when a Veteran/survivor is eligible for benefits under
 * both Compensation Service and PFS programs simultaneously.
 *
 * Indicators from the data:
 * - PensionType.dualEntitlement flag on the rating
 * - PensionType.compensationGreaterBenefit flag
 * - Presence of both SC rating issues AND pension award lines
 * - Basic eligibility decisions containing both comp grants and pension grants/denials
 */
private DualEntitlementStatus resolveDualEntitlement(
        String awardType,
        Set<String> awardLineTypes,
        boolean hasRatingProfile,
        Boolean dualEntitlementFlag,
        Boolean compensationGreaterBenefitFlag,
        List<String> basicEligibilityDecisionCodes) {

    // Only CPL and death types can have dual entitlement
    if (!AwardType.cplCode.equals(awardType)
        && !AwardType.cpdsCode.equals(awardType)) {
        return DualEntitlementStatus.NONE;
    }

    // Check the explicit dual entitlement flag from the pension rating
    if (Boolean.TRUE.equals(dualEntitlementFlag)) {
        if (Boolean.TRUE.equals(compensationGreaterBenefitFlag)) {
            return DualEntitlementStatus.PENSION_DENIED_COMP_GREATER;
        }
        return DualEntitlementStatus.COMP_DENIED_PENSION_GREATER;
    }

    // Check for mixed award lines as a secondary indicator
    boolean hasPensionLines = containsPensionAwardLines(awardLineTypes);
    boolean hasCompIndicators = hasRatingProfile;
    if (hasPensionLines && hasCompIndicators) {
        return DualEntitlementStatus.COMP_PENSION_ELECTION;
    }

    // Check basic eligibility decisions for pension grant + comp entitlement
    if (basicEligibilityDecisionCodes != null) {
        boolean hasPensionGrant = basicEligibilityDecisionCodes.stream()
            .anyMatch(code -> Arrays.asList("PGVA65", "PGVNH", "PGVDDSS", "EB").contains(code));
        boolean hasPensionDenial = basicEligibilityDecisionCodes.stream()
            .anyMatch(code -> Arrays.asList("NESP", "NWB").contains(code));
        if ((hasPensionGrant || hasPensionDenial) && hasCompIndicators) {
            if (hasPensionDenial) return DualEntitlementStatus.PENSION_DENIED_COMP_GREATER;
            return DualEntitlementStatus.COMP_PENSION_ELECTION;
        }
    }

    return DualEntitlementStatus.NONE;
}
```

#### Enhanced Letter Route Resolver — Dual Entitlement Routing

```java
/**
 * Dual entitlement routing.
 *
 * SCENARIO A: Comp is the greater benefit
 *   Primary letter: COMP_RADL
 *   Embedded PFS content:
 *     - Pension denial explanation
 *     - Pension rate that would have applied
 *     - Right to elect pension in the future
 *     - Income/expense verification requirements
 *
 * SCENARIO B: Pension is the greater benefit
 *   Primary letter: PFS_ADL
 *   Embedded Comp content:
 *     - SC rating decisions (still on record)
 *     - Combined evaluation percentage
 *     - Right to revert to comp if circumstances change
 *
 * SCENARIO C: Election pending
 *   Primary letter: COMP_RADL (with election notice)
 *   The Veteran must be informed of both rates and asked
 *   to elect the greater benefit.
 */
private LetterRoute resolveDualEntitlementRoute(ClaimFingerprint fingerprint) {
    switch (fingerprint.getDualEntitlement()) {
        case PENSION_DENIED_COMP_GREATER:
            // Comp is greater — RADL is primary, but with PFS pension denial sections
            return LetterRoute.COMP_RADL_WITH_PENSION_DENIAL;

        case COMP_DENIED_PENSION_GREATER:
            // Pension is greater — PFS ADL is primary, with comp rating info embedded
            return LetterRoute.PFS_ADL_WITH_COMP_RATING;

        case COMP_PENSION_ELECTION:
            // Election needed — RADL with election notice
            return LetterRoute.COMP_RADL_WITH_ELECTION_NOTICE;

        case DIC_PENSION_ELECTION:
            // Survivor dual entitlement
            return LetterRoute.COMP_RADL_WITH_PENSION_DENIAL;

        default:
            return LetterRoute.COMP_RADL;
    }
}
```

#### Enhanced LetterRoute Enum with Content Helpers

```java
public enum LetterRoute {
    PFS_ADL("PFS Automated Decision Letter"),
    COMP_RADL("Compensation Redesigned Automated Decision Letter"),
    BURIAL_LETTER("Burial Compensation Letter"),
    NRHLR_DECISION("Non-Rating Higher Level Review Decision Letter"),
    FEE_ALLOCATION_NOTICE("Fee Allocation Notice Letter"),
    NO_LETTER("No automated letter generated"),

    // ─── Dual Entitlement Composite Routes ───
    COMP_RADL_WITH_PENSION_DENIAL(
        "RADL with embedded pension denial — comp is the greater benefit"),
    PFS_ADL_WITH_COMP_RATING(
        "PFS ADL with embedded comp rating info — pension is the greater benefit"),
    COMP_RADL_WITH_ELECTION_NOTICE(
        "RADL with benefit election notice — Veteran must choose");

    private final String description;
    LetterRoute(String description) { this.description = description; }
    public String getDescription() { return description; }

    /** Returns true if this route requires PFS letter content sections */
    public boolean requiresPfsContent() {
        return this == PFS_ADL
            || this == PFS_ADL_WITH_COMP_RATING
            || this == COMP_RADL_WITH_PENSION_DENIAL;
    }

    /** Returns true if this route requires Comp/RADL content sections */
    public boolean requiresCompContent() {
        return this == COMP_RADL
            || this == COMP_RADL_WITH_PENSION_DENIAL
            || this == COMP_RADL_WITH_ELECTION_NOTICE
            || this == PFS_ADL_WITH_COMP_RATING;
    }
}
```

### Complete Comp + Pension Overlap Mapping Table

| Scenario | Award Lines | Rating? | Dual Ent. Flag | Fingerprint | → Letter Route |
|---|---|---|---|---|---|
| Veteran has comp only, no pension claim | SC lines only | Yes | No | `COMP:VET:REC:SC:NO_FID:ORIG:NONE` | **COMP RADL** |
| Veteran applies for pension, denied (comp is greater) | SC lines + pension denial | Yes | Yes (comp greater) | `COMP:VET:REC:MIXED:NO_FID:ORIG:PENSION_DENIED_COMP_GREATER` | **COMP RADL + Pension Denial** |
| Veteran applies for pension, granted (pension is greater) | IP line replaces SC lines | Yes | Yes (pension greater) | `PENSION:VET:REC:MIXED:NO_FID:ORIG:COMP_DENIED_PENSION_GREATER` | **PFS ADL + Comp Rating** |
| Veteran applies for pension, election pending | SC lines + IP line | Yes | Yes | `COMP:VET:REC:MIXED:NO_FID:ORIG:COMP_PENSION_ELECTION` | **COMP RADL + Election Notice** |
| Veteran has pension only, no comp history | IP/OLP/306P only | No | No | `PENSION:VET:REC:NSC:NO_FID:ORIG:NONE` | **PFS ADL** |

### Why Booleans Cannot Solve This

The current system has one boolean: `isPfsAdlLetter`. This gives you **two states**:
- `true` → PFS ADL
- `false` → RADL

But the comp+pension overlap requires **five distinct states** (the five rows above). The fingerprint approach captures all five because:

1. **`ServiceConnection.MIXED`** detects the overlap condition
2. **`DualEntitlementStatus`** classifies which flavor of overlap
3. **`LetterRoute` composite values** tell the template engine exactly what sections to include

The correspondence templates already have the building blocks — PFS decision point populators and RADL templates exist separately. The fingerprint provides the **intelligent orchestration layer** that decides which pieces to assemble for each unique claim state.

---

## Next Steps

1. **Unit tests** for `ClaimFingerprintExtractor` covering every `AwardType` code combination
2. **Unit tests** for `DualEntitlementStatus` resolution covering all comp+pension overlap scenarios
3. **Integration into `AuthorizeAwardLogic.confirmLetterOrFinalizeAward()`** to replace the `isEligibleForPfsAdl` boolean chain
4. **VBMS-Correspondence integration** — pass the `LetterRoute` enum through to `ManifestBuilder_PFS` vs. the existing RADL manifest builder
5. **Feature flag migration** — `isPfsAdlEnabled` becomes a route-level feature gate rather than a scattered boolean
6. **Composite template assembly** — implement `requiresPfsContent()` and `requiresCompContent()` hooks in the correspondence template engine to support dual entitlement letter routes
7. **Overlap handling** for additional edge cases (e.g., concurrent comp + pension awards on the same CPL, survivor DIC + death pension elections)