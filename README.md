# CarOnSale


CarOnSale uses a Webflow Registration Form on their website. This registration form sends all form submits on a custom API endpoint: https://app.caronsale.de/api/v1/user-registration

Because of form validation with ReCaptcha, the button submit button has some custom attributes:
data-sitekey="[REDACTED]"
data-callback="onSubmit"
data-action="submit"

So the ReCaptcha waits for the onSubmit function, before it can get send.

Therefore we have minimum code to work:

<script>
function onSubmit(token) {
    var form = document.getElementById('wf-form-Sign-Up');
    if (form) {
        form.submit();  //  This sends the form when the reCAPTCHA is validated.
    } else {
        console.error('Form not found');  // Error message if the form is not found.
    }
}
</script>

___________________________________


But this opens our next problem:

Because of the custom submit logic for the custom recaptcha, the form will be always send, when the recaptcha validation is set, even when no fields are filled. Thats why we need a custom logic to verify, that all fields are filled, before sending. thats why we extend our onSubmit function. For UX reasons, CarOnSales wishes to
1. color the not filled fields
2. have an clear error message (field placeholder + translated error message as a javascript alert)
3. in the right language

We also need to account for having multiple Forms on our websites. Lets say we have wf-form-Sign-Up and wf-form-Sign-Up-2. That is why we call the function at the end independently for both instances.

But we also need to take into account, that Webflow has a charakter limitation on sitewide code, and we already have much custom code on some websites. that is why we need to move the code in the head section. But for that we need to ensure that all the HTML elements have to be loaded. Thats why we wrap our calling function in DOMContentLoaded Listener

But we also need to ensure, that our Code works properly with the selected weglot language, thats why we set our weglot language in an sessionStorage Item

Furthermore we want to disable all not useable dropdown items:

So we have the following code:


<script>

  // make first option not clickable 
$("select option:first-child").attr("disabled", "true");
</script>

<script>

  // Get current user weglot settings
Weglot.on("switchersReady", function(initialLanguage) {
  var currentlan = Weglot.getCurrentLang();
	console.log(currentlan);
  sessionStorage.setItem("reg", currentlan);
});

</script>

<script>
  // make sure all HTML is loaded
document.addEventListener("DOMContentLoaded", function() {
    function onSubmit(token) {
     

        // Function for validating and handling forms
        function handleForm(formId) {
            var form = document.getElementById(formId);
            if (!form) {
               
                return; // If the form does not exist, exit the function
            }
          

            var requiredFields = form.querySelectorAll('[required]');
            var userLang = sessionStorage.getItem("reg") || "en";
          var errorMessages = {
    "at": {"input": "muss ausgefüllt werden", "checkbox": "Alle Häkchen sollen gesetzt werden"},
    "dk": {"input": "skal udfyldes", "checkbox": "Alle afkrydsningsfelter skal markeres"},
    "cz": {"input": "musí být vyplněno", "checkbox": "Všechny zaškrtávací políčka musí být zaškrtnuta"},
    "se": {"input": "måste fyllas i", "checkbox": "Alla kryssrutor måste vara ikryssade"},
    "de": {"input": "muss ausgefüllt werden", "checkbox": "Alle Häkchen sollen gesetzt werden"},
    "fr": {"input": "doit être rempli", "checkbox": "Toutes les cases doivent être cochées"},
    "nl": {"input": "moet worden ingevuld", "checkbox": "Alle selectievakjes moeten worden aangevinkt"},
    "it": {"input": "deve essere compilato", "checkbox": "Tutte le caselle devono essere selezionate"},
    "pl": {"input": "musi być wypełnione", "checkbox": "Wszystkie pola wyboru muszą być zaznaczone"},
    "tr": {"input": "doldurulmalıdır", "checkbox": "Tüm onay kutuları işaretlenmelidir"},
    "ru": {"input": "должно быть заполнено", "checkbox": "Все флажки должны быть установлены"},
    "hu": {"input": "ki kell tölteni", "checkbox": "Minden jelölőnégyzetet be kell jelölni"},
    "lt": {"input": "privaloma užpildyti", "checkbox": "Visi žymimieji laukeliai turi būti pažymėti"},
    "ro": {"input": "trebuie completat", "checkbox": "Toate căsuțele trebuie bifate"},
    "sl": {"input": "je treba izpolniti", "checkbox": "Vsa potrditvena polja je treba označiti"},
    "es": {"input": "debe ser completado", "checkbox": "Todas las casillas deben estar marcadas"},
    "pt": {"input": "deve ser preenchido", "checkbox": "Todas as caixas de seleção devem ser marcadas"},
    "en": {"input": "is required", "checkbox": "All checkboxes must be checked"}
    
};

          var allFilled = true;
    // Select all required fields within the form
    var requiredFields = form.querySelectorAll('[required]');
    // Retrieve user's language preference from sessionStorage, default to English
    var userLang = sessionStorage.getItem("reg") || "en";

    // Remove 'required-field-empty' class on change and handle checkbox-specific logic
    requiredFields.forEach(function(field) {
        field.classList.remove("required-field-empty");
        field.addEventListener('change', function() {
            if (this.type === "checkbox") {
                var labelSpan = this.nextElementSibling;
                if (labelSpan && labelSpan.classList.contains("w-form-label")) {
                    labelSpan.classList.remove("required-field-empty");
                }
            } else {
                this.classList.remove("required-field-empty");
            }
        });
    });

    // Initialize an array to collect error messages
    var errorMessageList = [];
    for (let field of requiredFields) {
        var isCheckbox = field.type === "checkbox";
        var isEmpty = isCheckbox ? !field.checked : !field.value;

        // Check if the field is empty and handle accordingly
        if (isEmpty) {
            allFilled = false;
            var errorMessage;

            if (isCheckbox) {
                var labelSpan = field.nextElementSibling;
                if (labelSpan && labelSpan.classList.contains("w-form-label")) {
                    labelSpan.classList.add("required-field-empty");
                }
                errorMessage = errorMessages[userLang]["checkbox"] || errorMessages["en"]["checkbox"];
            } else {
                field.classList.add("required-field-empty");
                if (field.tagName === "SELECT" && !field.placeholder) {
                    errorMessage = field.options[0].text + " " + (errorMessages[userLang]["input"] || errorMessages["en"]["input"]);
                } else {
                    errorMessage = (field.placeholder || field.name) + " " + (errorMessages[userLang]["input"] || errorMessages["en"]["input"]);
                }
            }
            errorMessageList.push(errorMessage);
        }
    }

    // Display all collected error messages if any field is not filled, otherwise submit the form
    if (!allFilled) {
        alert(errorMessageList.join("\n"));
    } else {
        form.submit();
    }
}

        // Handle specific forms based on their ID
        handleForm("wf-form-Sign-Up");
        handleForm("wf-form-Sign-Up-2");
    }

    // Make the onSubmit function globally available for ReCAPTCHA
    window.onSubmit = onSubmit;
});
</script>

_____________________________

